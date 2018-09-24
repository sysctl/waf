# waf
waf
#!/usr/bin/env bash
# Internet Gateway line failover script.
# @2018 WangMiao
# cron version

VERSION=1.0

usage() {
    echo "This script MUST BE run with root privileges."
}

if [ `id -u` != 0 ]
then
    usage
    exit 1
fi

CONFDIR="/root/lsw"
THRESHOLD=10
COOLDOWNDELAY=30
MARS_TARGET="1111"

MAX_LATENCY=2
TEST_COUNT=2
COOLDOWNDELAY=30

tel_list=${CONFDIR}/tel.list
tel_gw=${CONFDIR}/tel.gw
tel_fails=${CONFDIR}/tel.fails
tel_route=${CONFDIR}/tel.route
tel_route_default=${CONFDIR}/tel.route_default

uni_list=${CONFDIR}/uni.list
uni_gw=${CONFDIR}/uni.gw
uni_fails=${CONFDIR}/uni.fails
uni_route=${CONFDIR}/uni.route
uni_route_default=${CONFDIR}/uni.route_default

ruj_list=${CONFDIR}/ruj.list
ruj_gw=${CONFDIR}/ruj.gw
ruj_fails=${CONFDIR}/ruj.fails
ruj_route=${CONFDIR}/ruj.route
ruj_route_default=${CONFDIR}/ruj.route_default

tel_dev="n2fs66"
uni_dev="n2zs66"
ruj_dev="n2zs77"

tel_default_gw="192.168.0.1"
uni_default_gw="192.168.1.1"
ruj_default_gw="192.168.2.1"

log () {
    TYPE="$1"
    MSG="$2"
    DATE=`date +%b\ %d\ %H:%M:%S`
    case "$TYPE" in
        "ERROR" )
                    log2syslog "$TYPE" "$TYPE $MSG"
                    ;;
        "DEBUG" )
                    if [ "$DEBUG" = "1" ]
                    then
                        if [ "$QUIET" = "0" ]
                        then
                            echo "$DATE" "$MSG"
                        fi
                        log2syslog "$TYPE" "$TYPE $MSG"
                    fi

                    ;;
        "INFO" )
                    if [ "$QUIET" = "0" ] && [ "$DEBUG" = "1" ]
                    then
                        echo "$DATE $MSG"
                    fi
                    log2syslog "$TYPE" "$TYPE $MSG"
                    ;;
    esac
}

log2mars () {
    SUBJECT="$1"
    BODY="$2"
    DATE=`date +%b\ %d\ %H:%M:%S`
    if [ ! -z "$MAIL_TARGET" ]
    then
        echo "$DATE - $BODY" | mail -s "$SUBJECT" "$MAIL_TARGET" &
    fi
}

log2syslog () {
    TYPE=`echo "$1" | awk '{print tolower($0)}'`
    MSG="$2"
    echo "$MSG" | logger -t "lineswitch" -p daemon."$TYPE"
}

display_header () {

    log INFO "---------------------------------------"
    log INFO " Duoyi Gateway Failover Script $VERSION"
    log INFO "---------------------------------------"
    log INFO " Max latency in ms: $MAX_LATENCY"
    log INFO " Threshold before failover: $THRESHOLD"
    log INFO " Tests per host: $TEST_COUNT"
    log INFO "------------------------------"
}

lines=("tel" "uni" "ruj")

for i in ${lines[@]}
do
    echo "Begin ${i^^} Line Checking...\n"

    cur_list_file=`eval echo '$'"$i"'_list'`
    cur_gw_file=`eval echo '$'"$i"'_gw'`
    cur_fails_file=`eval echo '$'"$i"'_fails'`
    
    cur_list=(`cat $cur_list_file`)
    cur_gw=`cat $cur_gw_file`
    cur_fails=`cat $cur_fails_file`

    for z in ${cur_list[@]}
    do
        if [ -z "$z" ]
        then
        log ERROR "No targets to check availability, targets file $cur_list empty?."
        exit 1
        fi
        ping -W 2 -c 2 $z >> /dev/null 2>&1
        if [ $? -ne 0 ]
        then
            if [ "$cur_fails" -lt "$THRESHOLD" ]
            then
                ((cur_fails++))
                echo $cur_fails > $cur_fails_file
            fi
        else
            if [ "$cur_fails" -gt "0" ]
            then
                ((cur_fails--))
                echo $cur_fails > $cur_fails_file
            fi
        fi
    done
done

gw_tel=`cat $tel_gw`; gw_uni=`cat $uni_gw`; gw_ruj=`cat $ruj_gw`
fails_tel=`cat $tel_fails`; fails_uni=`cat $uni_fails`; $fails_ruj=`cat $ruj_fails`

declare -A linesgw
linesgw=([tel]="$gw_tel" [uni]="$gw_uni" [ruj]="$gw_ruj")

declare -A linestate
linestate=([tel]="$fails_tel" [uni]="$fails_uni" [ruj]="$fails_ruj")

for f in ${lines[@]}
do
    cur_l=`eval echo '$'"$f"`
    cur_l_file=`eval echo '$gw_'"$cur_l"`
    if [ ${linestate[$cur_l]} -ge "$THRESHOLD" ]
    then
        for bak_l in ${!linestate[@]}
        do
            if [ "${linestate[$bak_l]}" -lt "$THRESHOLD" ] && [ "$bak_l" == "${linesgw[$bak_l]}" ]
            then
                switch $cur_l $bak_l failover
                echo $bak_l > $cur_l_file
                MSG=" $cur_l link fails. change to $bak_l link."
                log2mars "$MSG"
                log INFO "$MSG"
                log DEBUG "Failover Cooldown started, sleeping for $COOLDOWNDELAY seconds."
                sleep "$COOLDOWNDELAY"
            fi
        done
    elif [ "${linestate[$cur_l]}" -eq "0" ] && [ "${linesgw[$cur_l]}" != "$cur_l" ]
    then
        switch ${linesgw[$cur_l]} $cur_l restore
        echo $cur_l > $cur_l_file
        MSG=" $cur_l link ok. restore to $cur_l link."
        log2mars "$MSG"
        log INFO "$MSG"
        log DEBUG "Failover Cooldown started, sleeping for $COOLDOWNDELAY seconds."
        sleep "$COOLDOWNDELAY"
    fi
done
        
switch () {
    before=$1
    after=$2
    s_type=$3

    PRE_CMD=${before}.CMD
    POST_CMD=${after}.CMD

    if [ ! -z "$PRE_CMD}" ]
    then
    eval "$PRE_CMD"
    fi

    if [ "$s_type" == "failover"]
    then
        gw=`eval echo '$'"$1"'_dev'`
        dft_gw=`eval echo '$'"$1"'_default_gw'`
        routelist=`eval echo '$'"$2"'_route'`

        for r in `cat $routelist`
        do 
            ip route chg $r dev $gw
            ip route chg default via $dft_gw 
        done


    elif [ "$s_type" == "restore" ]
    then
        curgw=`eval echo '$'"$1"`
        defgw=`eval echo '$'"$2"`
        routelist=`eval echo '$'"$2"'_route_default'`
        for r in `cat $routelist`
        do
            ip route chg $r
        done
        if [ $2 == "tel" ]
        then
            ip route chg default via "$tel_default_gw"
        fi
    fi

    if [ ! -z "$POST_CMD}" ]
    then
    eval "$POST_CMD"
    fi
} 
