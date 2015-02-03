
#VITESS目录结构#

`
vitess是google旗下youtube使用的分布式关系数据库中间件，由go语言开发，主要使用google内部的
mysql分支，本轮代码分析主要是为了开发mogujie自己的mysql水平sharding的组件DRDB。
`
##bin##
zksrv.sh 分布式锁服务zookeeper的服务端配置脚本；
###源码分析###
$ cat zksrv.sh 

`
zksrv.sh 脚本执行需要带三个参数，第一个是日志目录，第二个是配置目录，第三个是进程文件目录
`
logdir="$1"
config="$2"
pidfile="$3"
classpath="$VTROOT/dist/vt-zookeeper-3.3.5/lib/zookeeper-3.3.5-fatjar.jar:/usr/local/lib/zookeeper-3.3.5-fatjar.jar:/usr/share/java/zookeeper-3.3.5.jar"
mkdir -p "$logdir"  `强制创建目录`
touch "$logdir/zksrv.log" `	强制创建zk的日志文件`

log() {
  now=`/bin/date`
  echo "$now $*" >> "$logdir/zksrv.log"
  return 0
}

for java in /usr/local/bin/java /usr/bin/java; do
  if [ -x "$java" ]; then
    break
  fi
done

if [ ! -x "$java" ]; then
  log "ERROR no java binary found"
  exit 1
fi

if [ "$VTDEV" ]; then
  # use less memory
  java="$java -client -Xincgc -Xms1m -Xmx32m"
else
  # enable hotspot
  java="$java -server"
fi


cmd="$java -DZOO_LOG_DIR=$logdir -cp $classpath org.apache.zookeeper.server.quorum.QuorumPeerMain $config"

start=`/bin/date +%s`
log "INFO starting $cmd"
$cmd < /dev/null &> /dev/null &
pid=$!

log "INFO pid: $pid pidfile: $pidfile"
if [ "$pidfile" ]; then 
  if [ -f "$pidfile" ]; then
    rm "$pidfile"
  fi
  echo "$pid" > "$pidfile"
fi

wait $pid
log "INFO exit status $pid: $exit_status"



##config##
##data##
##dist##
##lib##
##pkg##
##py-vtdb##
##src##
##vtdataroot##
##vthook##
