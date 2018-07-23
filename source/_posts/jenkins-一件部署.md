---
title: jenkins  一件部署
date: 2018-07-20 16:21:56
tags: [jenkins]
categories: [jenkins]
---
## jenkins shell 命令添加

``` 
/usr/share/maven/bin/mvn clean install -U -Dmaven.test.skip=true -e 
sshpass -p 'admin' scp /data/jenkins/workspace/terminal_customer_test/target/terminal-customer-0.0.1-SNAPSHOT.jar admin@10.213.4.96:~/webapps
sshpass -p 'admin' ssh admin@10.213.4.96 'sh bin/webservices_ctl_spring_boot.sh restart'
```

## Spring Boot 一键启动脚本
```
#!/bin/sh

LANG=en_US.UTF-8
source /etc/profile

basedir=/home/admin
bindir=$basedir/bin
logsdir=$basedir/logs
ctl_pid=$basedir/bin/webctl.pid


#Web服务必须运行在admin账户下，非root
if [[ $UID -ne 500 ]]
then
    echo -e "Users must execute the $0 \033[31madmin\033[0m"
    exit 2
fi


#运行锁
#增加是否已运行webservices_ctl_spring_boot.sh的检测，避免多个操作同时进行出现的冲突
if [[ -e $ctl_pid ]]
then
    pid_v=`cat $basedir/bin/webctl.pid`
    if [[ $pid_v -gt 0 ]]
    then
        ps -ef | grep -w $pid_v | grep -Ev 'grep|ps -ef' >/dev/null 2>&1
        if [[ $? -eq 0 ]]
        then
            echo -e "\033[31m$0 exist, exit...\033[0m"
            exit 2
        else
            echo $$ > $ctl_pid
        fi
    fi
else
    echo $$ > $ctl_pid
fi


#启动webservice
start() {
    #去除maintenance状态
    if [[ -e $bindir/maintenance_file ]]
    then
        rm -f $bindir/maintenance_file 
    fi

    #检测Spring Boot是否已在运行，避免启动冲突
    #java_service_pid=`netstat -nltp | awk '$4~/8080$/&&$NF~/java/{gsub(/\/.*/,"",$NF);print $NF}'`
    java_service_pid=`ss -lp sport == :8080 | awk '$NF~/^users/{gsub(/\,/," ",$NF);print $NF}' | awk '{print $2}'`
    if [[ -n $java_service_pid ]]
    then
        ps -ef | grep -Ev 'grep|ps -ef' | grep $java_service_pid
        echo -e "\033[31mjava web service exist, exit...\033[0m"
        exit 2
    else
        echo -e "\033[32mplease waiting start java web service...\033[0m"
    fi

    #获取Spring Boot的路径，用作jar启动的参数，不同于Tomcat有固定的启动脚本，Spring Boot需要提供包的路径参数
    jar_path=`find /home/admin/webapps/ -maxdepth 1 -type f -name '*.jar'`
    if [[ -z $jar_path ]]
    then
        echo -e "\033[31mjar not find, exit...\033[0m"
        exit 2
    fi
    
    #根据主机名来获取服务所运行的环境：prod test dev，并将环境参数赋予Spring Boot的启动参数
    profiles=`hostname | awk -F '-' '{gsub(/[0-9][0-9]/,"",$NF);print $NF}'`
    if [[ $profiles == 'p' ]]
    then
        profiles='prod'
    elif [[ $profiles == 't' ]]
    then
        profiles='test'
    elif [[ $profiles == 'd' ]]
    then
        profiles='dev'
    fi

    #依据 OS Memory 来动态调整 JVM HeapMemory
    #降低Eden_Space_Size，让young  gc比较频繁，要短平快，把更多的内存空间留给PS_Old_Gen区域
    #-Xms 和 -Xmx 用于控制整个HeapMemory的大小， -Xmn 控制Eden_Space_Size，除去Eden_Space_Size剩下的空间基本都在PS_Old_Gen
    memtotal=`free -m | awk '$1=="Mem:"{print $2}'`
    if [[ $memtotal -gt 15800 ]]
    then
        export JAVA_OPTS="-server -Xms13312M -Xmx13312M -Xmn1024m -XX:+DisableExplicitGC"
    elif [[ $memtotal -gt 7800 ]]
    then
        export JAVA_OPTS="-server -Xms5632M -Xmx5632M -Xmn512m -XX:+DisableExplicitGC"
    elif [[ $memtotal -gt 3800 ]]
    then
        export JAVA_OPTS="-server -Xms2048M -Xmx2048M -Xmn512m -XX:+DisableExplicitGC"
    elif [[ $memtotal -gt 1800 ]]
    then
        export JAVA_OPTS="-server -Xms1024M -Xmx1024M -Xmn256m -XX:+DisableExplicitGC"
    elif [[ $memtotal -gt 800 ]]
    then
        export JAVA_OPTS="-server -Xms512M -Xmx512M -Xmn128m -XX:+DisableExplicitGC"
    fi

    #启动java web service, 通过/health来捕获应用是否启动成功
    #给java预设20s的启动时间，20s后每隔5是做一次http check，当服务启动时长超过120s时，则认为服务异常，需要人为介入
    mv -f $logsdir/spring.log $logsdir/spring.log_last
    java $JAVA_OPTS -jar $jar_path  --spring.profiles.active=$profiles --server.port=8080 --management.security.enabled=false \
                                   --server.tomcat.accesslog.enabled=true --server.tomcat.accesslog.directory=$logsdir \
                                   --server.tomcat.accesslog.pattern='%t [%I] %{X-Forwarded-For}i %a %r %s %b %{Referer}i "%{User-Agent}i" %D' >$logsdir/spring.log 2>&1 &
    sleep 20

    for ((i=1; i<21; i++))
    do
        #http_check=`/usr/bin/curl -s http://127.0.0.1:8080/health | /usr/bin/jq '.status' | awk '{gsub(/\"/,"",$0);print $0}'`
        http_check=`/usr/bin/curl -s http://127.0.0.1:8080/health`
        http_code=`echo -e $http_check | /usr/bin/jq '.status' | awk '{gsub(/\"/,"",$0);print $0}'`
        if [[ $http_code == UP ]]
        then
            echo -e "\033[32mjava service start up\033[0m"
            break
        elif [[ $http_code == DOWN ]]
        then
            echo -e "\033[31mjava service start down\033[0m"
            echo -e $http_check | /usr/bin/jq '.'
            stop
            break
        fi

        sleep 5

        if [[ $i == 20 ]]
        then
            stop
            echo -e "\033[31mjava service start time > 120s, exit...\033[0m"
            cat $logsdir/spring.log | grep -E -i '[0-9]+-[0-9]+-[0-9]+ [0-9]+:[0-9]+:[0-9]+[,.][0-9]+ ERROR' -B10 -A40 >> $bindir/result.error
            exit 2
        fi
    done
}


#关闭webservice
#从容关闭java web service，调用kill而非kill -9，留给java web service 30s的从容关闭时间，超过则强制kill
stop() {
    java_service_pid=`ps -ef | grep DisableExplicitGC | grep -v grep | awk '{print $2}'`    
    if [[ $java_service_pid -gt 0 ]]
    then
        kill $java_service_pid
        if [[ $? -eq 0 ]]
        then
            for ((i=1; i<31; i++))
            do
                java_service_pid=`ps -ef | grep DisableExplicitGC | grep -v grep | awk '{print $2}'`
                if [[ -z $java_service_pid ]]
                then
                    echo -e "\033[31mkill java web service done\033[0m"
                    break
                fi

                sleep 1

                if [[ $i == 30 ]] 
                then
                    kill -9 $java_service_pid
                    echo -e "\033[31mStrong kill java web service...\033[0m"
                fi
            done
        else
            echo -e "\033[31mkill java web service error, exit...\033[0m"
            exit 2
        fi
    else
        echo -e "\033[31mjava web service not exist, skip...\033[0m"
    fi

}


#重启webservice
restart() {
    stop
    sleep 1
    start
}


#维护webservice
#避免auto_abnormal_recover的自动异常恢复，在故障隔离或停机时使用
maintenance() {
    touch $bindir/maintenance_file 
    stop
}


#发布webservice
#只支持jar包发布，不支持增量发布，停机发布，非热部署
#先把要发布的jar包拷贝到release_dir，与webapps对比，做 jar包冲突检测（ip和jar的对应关系错误） 和 jar包MD5检测（打包失败或没有打包）
#检测无误后，执行stop（java发布需要先关闭）、backup（回滚需要）、start（启动服务完成变更）的发布逻辑
#最后，保留最近5次的backup_jar，多余的备份将会删除，避免太多的备份数量导致磁盘空间异常
release() {
    
    #创建release_dir，发布时会将jar包暂存在此目录
    release_dir=$basedir/release_dir
    mkdir -p $release_dir

    export RSYNC_PASSWORD="admin"
    num=0
    while [ 1 ]
    do
        ((num++))
        if [[ $num -ge 3 ]]
        then
            echo -e "\033[31mjenkis workspace no such file or dir: $jarfile , release error, exit...\033[0m"
            exit 2
        else
            /usr/bin/rsync -CavzP ystops@ci.yst.com.cn::workspace/$jarfile $release_dir/
            if [[ $? -eq 0 ]]
            then
                break
            fi
        fi
    done


    #jar包冲突检测 jar包MD5检测
    release_jar=$release_dir/${jarfile##*/}
    webapps_jar=`ls -l $basedir/webapps | awk '$NF~/.*jar$/{print $NF}'`
    webapps_jar=$basedir/webapps/$webapps_jar

    #判断要发布的jar包是否在release_dir和webapps中都存在
    #存在时，判断release_dir和webapps中的jar_name是否相同，相同则可以继续发布，不同则意味着ip和jar的对应关系错误，退出发布
    #当jar_name相同时，检查两个jar包的MD5是否相同，相同则意味着打包失败或没有打包，发布不会带来新的代码，退出发布
    if [[ -f $release_jar ]] && [[ -f $webapps_jar ]]
    then
        if [[ ${release_jar##*/} != ${webapps_jar##*/} ]]
        then
            echo -e "\033[31mrelease_jar_name: ${release_jar##*/} != webapps_jar_name: ${webapps_jar##*/} , please check , release exit...\033[0m"
            rm -f $release_jar
            exit 2
        fi

        release_jar_md5=`/usr/bin/md5sum $release_jar | awk '{print $1}'`
        webapps_jar_md5=`/usr/bin/md5sum $webapps_jar | awk '{print $1}'`
        if [[ $release_jar_md5 == $webapps_jar_md5 ]]
        then
            echo -e "\033[31m$release_jar\t MD5 $release_jar_md5\033[0m"
            echo -e "\033[31m$webapps_jar\t MD5 $webapps_jar_md5\033[0m"
            echo -e "\033[31mrelease_jar_md5 == webapps_jar_md5 , please repackaging , release exit...\033[0m"
            rm -f $release_jar
            exit 2
        fi
    elif [[ -f $release_jar ]] && ! [[ -f $webapps_jar ]]
    then
        echo -e "\033[32mwebapps_jar is empty, frits relese...\033[0m"
    fi


    #backup
    #发布前的备份
    #调整了 stop 和 backup 的执行顺序，要力保发布是可以回滚的
    mkdir -p $basedir/backup/$today
    #backfile=`find $basedir/webapps -name '*.jar'`
    backfile=`ls -l $basedir/webapps | awk '$NF~/.*jar$/{print $NF}'` 
    if [[ -n $backfile ]]
    then
        for i in $backfile
        do
            cp -p $basedir/webapps/$i $basedir/backup/$today/${i##*/}.$timestamp
            if [[ $? -eq 0 ]]
            then
                echo -e "\033[32m${i##*/} backup successful\033[0m"
            else
                echo -e "\033[31mrelease backup error, release exit...\033[0m"
                exit 2
            fi
        done
    else
        echo -e "\033[32mfrist release, no backup...\033[0m"
    fi

    #关闭webservice
    stop

    #执行release
    rm -rf $basedir/webapps/*
    mv $release_jar $basedir/webapps/

    #启动webservice，完成变更
    start

    #backup_jarfile clean
    #保留最近5次的backup_jar，多余的备份将会删除
    backup_jarfile_count=`ls -l $basedir/backup | awk '$1~/^d/' | wc -l`
    last_backup_num=`awk 'BEGIN{printf"%.0f\n", '$backup_jarfile_count' - 5}'`
    if [[ $last_backup_num -gt 0 ]]
    then
        cd $basedir/backup
        ls -l | awk '$1~/^d/' | head -n $last_backup_num | awk '{print "rm -rf "$NF}' | sh
    fi

}


#回滚webservice
#增加回滚jar包是否存在、是否与当前jar包的md5值一致的判断
#执行过程：check、stop、rollback、start
rollback () {

    jarfile=`echo $backup_jarfile | awk '{gsub(/.*\//,"",$NF);gsub(/\.jar\.[0-9]+/,".jar",$NF);print $NF}'`
    jarfile=$basedir/webapps/$jarfile
    backup_jarfile=$basedir/backup/$backup_jarfile

    #判断回滚jar包是否存在，避免提供错误的回滚包，导致回滚失败，还删除了webapps
    #判断回滚jar包是否与webapps的MD5值相同，相同则意味着回滚不会有任何的收益，需要确认
    if [[ -f $backup_jarfile ]] && [[ -f $jarfile ]]
    then
        rollback_jarfile=`ls -l $backup_jarfile | awk '{gsub(/.*\//,"",$NF);gsub(/\.jar\.[0-9]+/,".jar",$NF);print $NF}'`
        if [[ $rollback_jarfile != ${jarfile##*/} ]]
        then
            echo -e "\033[31mrollback_jar_name: $backup_jarfile != webapps_jar_name: ${jarfile##*/} , please check , rollback exit...\033[0m"
            exit 2
        fi
                                                                                                               
        rollback_jar_md5=`/usr/bin/md5sum $backup_jarfile | awk '{print $1}'`
        webapps_jar_md5=`/usr/bin/md5sum $jarfile | awk '{print $1}'`
        if [[ $rollback_jar_md5 == $webapps_jar_md5 ]]
        then
            echo -e "\033[31m$backup_jarfile\t MD5 $rollback_jar_md5\033[0m"
            echo -e "\033[31m$jarfile\t MD5 $webapps_jar_md5\033[0m"
            echo -e "\033[31mrollback_jar_md5 == webapps_jar_md5 , please check , rollback exit...\033[0m"
            exit 2
        fi
    else
        echo -e "\033[31mbackup_dir no such file or dir: $backup_jarfile , rollback error, exit...\033[0m"
        exit 2
    fi

    #关闭webservice
    stop

    #执行rollback
    rm -rf $basedir/webapps/*
    cp -p $backup_jarfile $jarfile

    #开启webservice，完成rollback
    start
}


case "$1" in
    start)
        $1
        ;;
    stop)
        $1
        ;;
    restart)
        $1
        ;;
    maintenance)
        $1
        ;;
    release)
        if [[ $# -eq 4 ]] && [[ $2 =~ '.jar'$ ]]
        then
            jarfile=$2
            today=$3
            timestamp=$4
        else
            echo -e "release must bring \033[31mjar path\033[0m, bye!"
            exit 2
        fi
        $1
        ;;
    rollback)
        if [[ $# -eq 2 ]] && [[ $2 =~ .*'.jar.'[0-9]+$ ]]
        then
            backup_jarfile=$2
        else
            echo -e "rollback must bring \033[31mbackup jar path\033[0m, bye!"
            exit 2
        fi
        $1
        ;;
    *)

    echo $"Usage: $0 {start|stop|restart|release|rollback|maintenance}"
    exit 2
esac



```


