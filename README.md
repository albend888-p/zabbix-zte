# zabbix-zte-sms-alert-send.sh

Custom bash script for sending SMS via local ZTE modem by zabbix triggers

```bash
#!/bin/bash

SCRIPT_DIR=$( cd -- "$( dirname -- "${BASH_SOURCE[0]}" )" &> /dev/null && pwd )

function encode() {
   printf "%s" "$*" | perl -pe 's/([^0-9a-zA-Z_])/sprintf("%%%02x",ord($1))/ge'
}

URL=$1
PASSWORD=$2

ARGS_1=$3
ARGS_2=$4
ARGS_3=$5

ZONE=$(date +'%z')
DATE=$(date +'%y;%m;%d;%H;%M;%S')
DATE+=";"
DATE+="$(echo "${ZONE}" | sed 's/[^+1-9]//g' )"

REFERER="$URL/index.html"
URL_SET="$URL/goform/goform_set_cmd_process"
URL_GET="$URL/goform/goform_get_cmd_process"
HASHED_PASSWORD=$(printf $PASSWORD | base64 -)
LOGIN_QUERY="isTest=false&goformId=LOGIN&password=$HASHED_PASSWORD"
COOKIE_STORE="$SCRIPT_DIR/cookie_zte"  

ENCODE_DATE="$(encode "${DATE}")"
#ENCODE_PHONE="$(encode "${ARGS_1}")"
ENCODE_PHONE="$(echo "${ARGS_1}" | sed 's/[^0-9]//g' )"
#ENCODE_SUBJECT="$(encode "${ARGS_2}")"
ENCODE_MESSAGE="$(echo $ARGS_3 | cut -c1-159 | iconv -f utf8 -t utf16be | xxd -pu -c 512)"

Authenticate() {
    IS_LOGGED=$(curl -b $COOKIE_STORE -s --header "Referer: $REFERER" $URL_GET\?multi_data\=1\&isTest\=false\&sms_received_flag_flag\=0\&sts_received_flag_flag\=0\&cmd\=loginfo | jq --raw-output .loginfo) 
    # Login
    if [ "$IS_LOGGED" == "ok" ]; then
        echo "😀 Logged in" 
        return 0
    else
        LOGIN=$(curl -c $COOKIE_STORE -X POST --header "Referer: $URL_GET" --header "X-Requested-With: XMLHttpRequest" --data "$LOGIN_QUERY" $URL_SET | jq --raw-output .result)
        echo "🤔 Loggining in to ZTE"		
	if [ "$LOGIN" == "0" ]; then
            echo "😀 success"
            return 0
        else
            echo "😡 error"
        fi
    fi
    return 1
}

Send_Messages() {
    RESULT=$(curl -b $COOKIE_STORE -s --header "Referer: $REFERER" $URL_SET\?isTest\=false\&goformId\=SEND_SMS\&notCallback\=true\&Number\=$ENCODE_PHONE\&sms_time\=$ENCODE_DATE\&MessageBody\=$ENCODE_MESSAGE\&ID\=-1\&encode_type\=UNICODE | jq --raw-output .result)
    echo "🤔 Sending SMS"		
    if [ "$RESULT" == "success" ]; then
	echo "😌 SMS sended"
	return 0
    else
	echo "😡 Could not send SMS"
	echo "👇 Encode message:"
	echo "$ENCODE_MESSAGE"
	exit 1
    fi
}


command -v jq >/dev/null 2>&1 || {
    echo >&2 "'jq' is required but not installed. Aborting."
    exit 1
}

if Authenticate; then
    Send_Messages
fi 
```

