大致分几个阶段完成。

1. 环境准备
1.1. golang，可参考 https://golang.org/doc/install
1.2. rust，大致流程：
    export RUST_VERSION=nightly-2020-03-12
    curl -sSf https://sh.rustup.rs | sh -s -- -y --default-toolchain "$RUST_VERSION" && rustc --version && cargo --version 

或者在已有的环境执行：

    rustup update nightly-2020-03-12

    rustup override set nightly-2020-03-12



2. 代码下载

2.1. fork https://github.com/tikv/tikv  and clone to local

2.2. fork https://github.com/pingcap/pd and clone to local

2.3. fork https://github.com/pingcap/tidb and clone to local



3. 编译

3.1. cd $TIKV_LOCAL_ROOT/ && make && cp target/release/tikv-server ./bin/

3.2. cd $TIPD_LOCAL_ROOT/ && make

3.3. cd $TIDB_LOCAL_ROOT/ && make



4. 启动

4.1. pd: /bin/pd-server --name=pd1 --data-dir=pd1 --client-urls="http://127.0.0.1:2379" --peer-urls="http://127.0.0.1:2380" --initial-cluster="pd1=http://127.0.0.1:2380" --log-file=pd1.log

4.2. tikv: 

期间碰到 umlimit open files 不够的情况，改为 root 用户执行

sudo ./bin/tikv-server --pd-endpoints="127.0.0.1:2379" --addr="127.0.0.1:20160" --data-dir=tikv1 --log-file=tikv1.log > /dev/null 2>&1 &

sudo ./bin/tikv-server --pd-endpoints="127.0.0.1:2379" --addr="127.0.0.1:20161" --data-dir=tikv2 --log-file=tikv2.log > /dev/null 2>&1 &
sudo ./bin/tikv-server --pd-endpoints="127.0.0.1:2379" --addr="127.0.0.1:20162" --data-dir=tikv3 --log-file=tikv3.log > /dev/null 2>&1 &
4.3. tidb
./bin/tidb-server --store=tikv --path=127.0.0.1:2379

5. 代码修改
修改 tidb 项目 store/tikv/txn.go 中的 StartTS() 方法，增加 logutil.BgLogger().Info("Hello transaction")。
详见： https://github.com/meiking/tidb/pull/1/files

6. 验证与测试
打开连接： http://127.0.0.1:2379/dashboard/#/cluster_info/instance 可查看集群实例。
image.png


创建好database和table后，运行

BEGIN;

update table1 set `value` = 'val0' where id = 2;

update table1 set `value` = 'val01' where id = 1;

insert into table1(`value`) value(' val x ');

COMMIT;

后，tidb stdout 中可见打印 "Hello transaction"。
