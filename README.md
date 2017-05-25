
## mha_consul

Auto failover mysql with [masterha_manager](https://github.com/yoshinorim/mha4mysql-manager/tree/master/bin) based on consul cluster to avoid network instability.

## Dependency

```
perl-libwww-perl
perl-JSON
```

## Background

We often don't use `masterha_manager` to make a auto mysql failover, because of the network instability(switch, cable or network card), for example, restart the host network card which the `masterha_manager` running on, then we know the MySQL is alive. From this aspect, we cannot determine whether MySQL master is dead or not depend only 1 check point.

Fortunately, We can use consul cluster features to check mysql health with multi point, when the problem check point greater than half consul servers, we think the MySQL master is abnormal. such as the following structure:
```

       <checkmysql>         <checkmysql>         <checkmysql>
            |                   |                     |
       +---------+          +---------+          +---------+
       | consul1 |          | consul2 |          | consul3 |
       +---------+          +---------+          +---------+
                  \             |               /
                   \            |              /
                    \           |             /
                     \          |            /
                     +----------------------+
                     |   consul cluster     |
                     +----------------------+
                                |
                                |
                     +----------------------+  watch
                     | consul-template      | -------> < mysqlxxx.tpl >  ---> <mysqlxxx.conf>
                     +----------------------+
                                                                                    |
                                                                                    | changed
                                                                                    |
                                                                        +--------------------------+  
                                                                        | masterha_manager_consul  |
                                                                        +--------------------------+

```

The `checkmysql` is deployed on every consul server, this means we have multi points to check whether the MySQL master is abnomal or not, if MySQL is ok, `checkmysql` will set the key `mysql/mysqlxxxx/node-consul` with value 1, otherwise set the value 0, where `node-consul` default value is the hostname that consul server running on.

After all `checkmysql` update consul keys, we use [consul-template](https://github.com/hashicorp/consul-template) to watch all the keys based on the template file `mysqlxxx.tpl`, consul-template will generate `mysqlxxx.conf` when any of the keys changed, then consul-template will invoke `masterha_manager_consul` to execute failover.

We have rewrite the `masterha_manager_consul`, the method `MHA::HealthCheck::wait_until_unreachable` will not running with infinite loop way, and it'll exit if there is less than half number consul servers check the MySQL is abnormal, oterhwise `masterha_manager_consul` will fork a child process to execute failover. 

## How to use

the full code structure:

```
mha_manager_consul
├── bin
│   ├── checkmysql
│   └── masterha_manager_consul
├── conf
│   ├── db.cnf
│   └── template-config
├── consul
│   ├── acl
│   │   ├── policy.ano
│   │   └── policy.key
│   └── conf.d
│       └── server.json
├── README.md
└── template
    └── mysql3308.tpl
```

test env:

|ip|os|hostname|version|
|:-:|:-:|:-:|:-|
|10.0.21.5|centos 6.5|cz-test1|consul 0.8v|
|10.0.21.7|centos 6.5|cz-test2|consul 0.8v|
|10.0.21.17|centos 6.5|cz-test3|consul 0.8v|

### Note

All the following steps are assume you have  installed consul cluster, and before run `checkmysql`, we must set consul acls to ensure that don't anyone access consul api directly.

The token params is the option `acl_master_token` which in you consul server configure file, the `policy.ano` is used to disable anonymous to access the related key `mysql/*`, and `policy.key` is used to enable the specifed token (`dcb5b583-cd36-d39d-2b31-558bebf86502`) to access the related key `mysql/*`. read more from [consul acl](https://www.consul.io/api/acl.html)

Using the following command to set acls:
```
#curl -X PUT --data @policy.ano http://localhost:8500/v1/acl/update?token=e95597e0-4045-11e7-a9ef-b6ba84687927
{"ID":"anonymous"}

#curl -X PUT --data @policy.key http://localhost:8500/v1/acl/update?token=e95597e0-4045-11e7-a9ef-b6ba84687927
{"ID":"dcb5b583-cd36-d39d-2b31-558bebf86502"}
```

### checkmysql

run the `checkmysql` on every consul server host, the token params is from above consul acl step, the tag option from the `db.cnf` configure file, which is the instance present:
```
perl checkmysql --conf db.cnf --verbose --tag mysql3308 --token dcb5b583-cd36-d39d-2b31-558bebf86502
```
#### note: if you have vip in MySQL master, the host option in `db.cnf` should be set the vip address.
 
### consul-template

`consul-template` watch the keys based on mysqlxxx.tpl, when the keys changed, it generate mysqlxxx.conf file. We use mysql3308.tpl to explain:

#### start consul-template

```
# consul-template -config config 
2017/05/25 10:11:13 [DEBUG] (logging) enabling syslog on LOCAL5
```

mysqlxxx.tpl template file:
```
# node3308
{{ range ls "mysql/mysql3308" }}
{{ .Key }}:{{ .Value }}{{ end }}
```

After keys changed, `consul-template` generate mysql3308.conf :
```
# node3308

cz-test1:1
cz-test2:1
cz-test3:1
```

if there is less than half consul servers check the MySQL is abnormal, consul-template will print the following message:
```
[2017-05-25T10:24:15] status ok, skip switch..
```
otherwise print error, and invoke `masterha_manager_consul`:
```
[2017-05-25T10:24:48] status error, need switch..
Wed May 24 10:24:48 2017 - [info] Reading default configuration from /etc/masterha/app_default.cnf..
...
...
```

### conf.d/server.json

We have set the `address` option to `consul.service.consul:8500`, why use dns item instead the ip address? because we think consul-template cann't connect to one consul server with the ip address when the network instability even `consul-template` have retry features, so we recommand set all consul server ip address as the consul dns entries. such as:
```
# dig @10.0.21.5 consul.service.consul
......
......
;; QUESTION SECTION:
;consul.service.consul.        IN    A

;; ANSWER SECTION:
consul.service.consul.    0    IN    A    10.0.21.7
consul.service.consul.    0    IN    A    10.0.21.5
consul.service.consul.    0    IN    A    10.0.21.17

```
any one occurs error, consul cluster will forward request to normal consul server.

## License

MIT / BSD
