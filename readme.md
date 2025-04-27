# compile && debug
cmake : -DWITH_BOOST=/Users/morphyzhang/Documents/Code/mysql-server/boost_1_77_0 -DWITH_SSL=/opt/homebrew/Cellar/openssl@3/3.3.2 -DWITH_DEBUG=1 -DCMAKE_INSTALL_PREFIX=build_out -DMYSQL_DATADIR=build_out/data -DSYSCONFDIR=build_out/etc -DMYSQL_TCP_PORT=3307 -DMYSQL_UNIX_ADDR=mysql-debug.sock  
select binfile `mysqld`: 
1. add Program arguments: --initialize-insecure. then will generate default user `root` with no password.
2. delete program arguments, then continue to debug.
mysql client :
`./mysql -uroot -P3307 -h127.0.0.1`

# Directories
1. build-out is cmdline cmake && make.
2. cmake-build-8039-debug is all using clion to build.