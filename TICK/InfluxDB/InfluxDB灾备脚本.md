DURATION=1800
STARTTIME=LASTENDTIME
ENDTIME-STARTTIME + DURATION

DBLIST=$(influx -execute 'show databases')

for DB in $DBLIST; do
  influx_inspect export -compress -database $DB -start $STARTTIME -end $ENDTIME out $DB-$ENDTIME.gz
  aws s3 move $DB-$ENDTIME.gz s3://<YOURBACKET>/EXPORT/
done

