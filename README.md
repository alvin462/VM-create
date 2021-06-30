# VM-create
常見的虛擬化術語:
* Host： 就是虛擬機器所在的那部實體主機
* VM： 就是虛擬機器硬體的意思
* Guest OS： 在 VM 上面所安裝的獨立的作業系統
* Client： 就是你的工作機，跟上述的環境無關！舉例來說，你目前在使用的這部 Windows 工作機，可以透過 remote-viewer 連線到 VM 上，那麼這個系統就可以稱為 client 端的系統。

經常使用到的軟體：
* KVM： 整合到 Linux 核心，是最重要的虛擬化技術，可以虛擬出 CPU 的硬體
* qemu： 相對於 KVM，qemu 則主要在虛擬出各項週邊設備，包括磁碟、網卡、USB、顯卡、音效等
* libvirtd： 提供使用者一個管理 VM 的服務
* virt-manager： 有點類似圖形界面，可以搭配 libvirtd 進行虛擬機器的管理。
* virsh： 終端機界面的管理指令。

## 安裝及啟動
```cmd
yum install qemu-kvm qemu-img virt-manager libvirt libvirt-client virt-install virt-viewer bridge-utils
systemctl start libvirtd
systemctl enable libvirtd
systemctl status libvirtd
● libvirtd.service - Virtualization daemon
   Loaded: loaded (/usr/lib/systemd/system/libvirtd.service; enabled; vendor pr>
   Active: active (running) since Thu 2021-07-01 02:17:48 CST; 1s ago
     Docs: man:libvirtd(8)
           https://libvirt.org
 Main PID: 83842 (libvirtd)

```
---

## 觀察目前啟動的VM

```
virsh list [--all]
 Id    名稱                         狀態
----------------------------------------------------

virsh net-list [--all]
 名稱               狀態     自動啟動  Persistent
----------------------------------------------------------
 default              啟用     yes           yes

```
預設不會有 VM ，而預設會啟用一個名為『 default 』的橋接器！提供一個 VM 內部可以連線的網路狀態！

---

## 關閉與取消定義的虛擬機器、橋接器等
在確認了有 default 這個橋接器，觀察Port狀態
```
netstat -tulnp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 192.168.122.1:53        0.0.0.0:*               LISTEN      5250/dnsmasq
udp        0      0 192.168.122.1:53        0.0.0.0:*                           5250/dnsmasq
udp        0      0 0.0.0.0:67              0.0.0.0:*                           5250/dnsmasq

```
* Port 53: 提供 VM 向外部查詢 DNS
* port 67: 是提供VMDHCP自動取得網路參數

---
 
## 觀察目前網路卡資訊
```
ip link show
virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500....
virbr0-nic: <BROADCAST,MULTICAST> mtu 1500....
```
發現出現了virbr0、virbr0-nic兩張介面卡。
* virbr0：會自動加入一個192.168.122.X/24的網段給虛擬機器使用，並且使用的是 NAT 的機制，因此使用這個 virbr0 橋接器連結的系統， 就可以自動的透過你的 host 上網了。

觀察設定檔察看預設網路設定值
由於網路轉遞是 NAT 技術，而 DHCP 的範圍則是 192.168.122.2 ~ 192.168.122.254 之間！ 至於 DHCP server 的 IP 則是 192.168.122.1 喔

```
vim /etc/libvirt/qemu/networks/default.xml
<network>
  <name>default</name>
  <uuid>d77a80ea-c1cf-44d1-9157-a71ce0f46b0a</uuid>
  <forward mode='nat'/>
  <bridge name='virbr0' stp='on' delay='0'/>
  <mac address='52:54:00:60:8e:8f'/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
    </dhcp>
  </ip>
</network>
```
由於
*:information_source: Especially on Linux, make sure your user has the [required permissions][linux-postinstall] to
interact with the Docker daemon.*

By default, the stack exposes the following ports:

* 5044: Logstash Beats input
* 5000: Logstash TCP input
* 9600: Logstash monitoring API
* 9200: Elasticsearch HTTP
* 9300: Elasticsearch TCP transport
* 5601: Kibana

**:warning: Elasticsearch's [bootstrap checks][booststap-checks] were purposely disabled to facilitate the setup of the
Elastic stack in development environments. For production setups, we recommend users to set up their host according to
the instructions from the Elasticsearch documentation: [Important System Configuration][es-sys-config].**

### SELinux

On distributions which have SELinux enabled out-of-the-box you will need to either re-context the files or set SELinux
into Permissive mode in order for docker-elk to start properly. For example on Redhat and CentOS, the following will
apply the proper context:

```console
$ chcon -R system_u:object_r:admin_home_t:s0 docker-elk/
```

### Docker for Desktop

#### Windows

Ensure the [Shared Drives][win-shareddrives] feature is enabled for the `C:` drive.

#### macOS

The default Docker for Mac configuration allows mounting files from `/Users/`, `/Volumes/`, `/private/`, and `/tmp`
exclusively. Make sure the repository is cloned in one of those locations or follow the instructions from the
[documentation][mac-mounts] to add more locations.

## Usage

### Version selection

This repository tries to stay aligned with the latest version of the Elastic stack. The `main` branch tracks the current
major version (7.x).

To use a different version of the core Elastic components, simply change the version number inside the `.env` file. If
you are upgrading an existing stack, please carefully read the note in the next section.

**:warning: Always pay attention to the [official upgrade instructions][upgrade] for each individual component before
performing a stack upgrade.**

Older major versions are also supported on separate branches:

* [`release-6.x`](https://github.com/deviantony/docker-elk/tree/release-6.x): 6.x series
* [`release-5.x`](https://github.com/deviantony/docker-elk/tree/release-5.x): 5.x series (End-Of-Life)

### Bringing up the stack

Clone this repository onto the Docker host that will run the stack, then start services locally using Docker Compose:

```console
$ docker-compose up
```

You can also run all services in the background (detached mode) by adding the `-d` flag to the above command.

**:warning: You must rebuild the stack images with `docker-compose build` whenever you switch branch or update the
version of an already existing stack.**

If you are starting the stack for the very first time, please read the section below attentively.

### Cleanup

Elasticsearch data is persisted inside a volume by default.

In order to entirely shutdown the stack and remove all persisted data, use the following Docker Compose command:

```console
$ docker-compose down -v
```

## Initial setup

### Setting up user authentication

*:information_source: Refer to [How to disable paid features](#how-to-disable-paid-features) to disable authentication.*

The stack is pre-configured with the following **privileged** bootstrap user:

* user: *elastic*
* password: *changeme*

Although all stack components work out-of-the-box with this user, we strongly recommend using the unprivileged [built-in
users][builtin-users] instead for increased security.

1. Initialize passwords for built-in users

    ```console
    $ docker-compose exec -T elasticsearch bin/elasticsearch-setup-passwords auto --batch
    ```

    Passwords for all 6 built-in users will be randomly generated. Take note of them.

1. Unset the bootstrap password (_optional_)

    Remove the `ELASTIC_PASSWORD` environment variable from the `elasticsearch` service inside the Compose file
    (`docker-compose.yml`). It is only used to initialize the keystore during the initial startup of Elasticsearch.

1. Replace usernames and passwords in configuration files

    Use the `kibana_system` user (`kibana` for releases <7.8.0) inside the Kibana configuration file
    (`kibana/config/kibana.yml`) and the `logstash_system` user inside the Logstash configuration file
    (`logstash/config/logstash.yml`) in place of the existing `elastic` user.

    Replace the password for the `elastic` user inside the Logstash pipeline file (`logstash/pipeline/logstash.conf`).

    *:information_source: Do not use the `logstash_system` user inside the Logstash **pipeline** file, it does not have
    sufficient permissions to create indices. Follow the instructions at [Configuring Security in Logstash][ls-security]
    to create a user with suitable roles.*

    See also the [Configuration](#configuration) section below.

1. Restart Kibana and Logstash to apply changes

    ```console
    $ docker-compose restart kibana logstash
    ```

    *:information_source: Learn more about the security of the Elastic stack at [Tutorial: Getting started with
    security][sec-tutorial].*

### Injecting data

Give Kibana about a minute to initialize, then access the Kibana web UI by opening <http://localhost:5601> in a web
browser and use the following credentials to log in:

* user: *elastic*
* password: *\<your generated elastic password>*

Now that the stack is running, you can go ahead and inject some log entries. The shipped Logstash configuration allows
you to send content via TCP:

```console
# Using BSD netcat (Debian, Ubuntu, MacOS system, ...)
$ cat /path/to/logfile.log | nc -q0 localhost 5000
```

```console
# Using GNU netcat (CentOS, Fedora, MacOS Homebrew, ...)
$ cat /path/to/logfile.log | nc -c localhost 5000
```

You can also load the sample data provided by your Kibana installation.

### Default Kibana index pattern creation

When Kibana launches for the first time, it is not configured with any index pattern.

#### Via the Kibana web UI

*:information_source: You need to inject data into Logstash before being able to configure a Logstash index pattern via
the Kibana web UI.*

Navigate to the _Discover_ view of Kibana from the left sidebar. You will be prompted to create an index pattern. Enter
`logstash-*` to match Logstash indices then, on the next page, select `@timestamp` as the time filter field. Finally,
click _Create index pattern_ and return to the _Discover_ view to inspect your log entries.

Refer to [Connect Kibana with Elasticsearch][connect-kibana] and [Creating an index pattern][index-pattern] for detailed
instructions about the index pattern configuration.

#### On the command line

Create an index pattern via the Kibana API:

```console
$ curl -XPOST -D- 'http://localhost:5601/api/saved_objects/index-pattern' \
    -H 'Content-Type: application/json' \
    -H 'kbn-version: 7.13.2' \
    -u elastic:<your generated elastic password> \
    -d '{"attributes":{"title":"logstash-*","timeFieldName":"@timestamp"}}'
```

The created pattern will automatically be marked as the default index pattern as soon as the Kibana UI is opened for the
first time.

## Configuration

*:information_source: Configuration is not dynamically reloaded, you will need to restart individual components after
any configuration change.*

### How to configure Elasticsearch

The Elasticsearch configuration is stored in [`elasticsearch/config/elasticsearch.yml`][config-es].

You can also specify the options you want to override by setting environment variables inside the Compose file:

```yml
elasticsearch:

  environment:
    network.host: _non_loopback_

```console
$ curl -XPOST -D- 'http://localhost:9200/_security/user/elastic/_password' \
    -H 'Content-Type: application/json' \
    -u elastic:<your current elastic password> \
    -d '{"password" : "<your new password>"}'
```

## Extensibility

### How to add plugins

To add plugins to any ELK component you have to:

1. Add a `RUN` statement to the corresponding `Dockerfile` (eg. `RUN logstash-plugin install logstash-filter-json`)
1. Add the associated plugin code configuration to the service configuration (eg. Logstash input/output)
1. Rebuild the images using the `docker-compose build` command

### How to enable the provided extensions

A few extensions are available inside the [`extensions`](extensions) directory. These extensions provide features which
are not part of the standard Elastic stack, but can be used to enrich it with extra integrations.

The documentation for these extensions is provided inside each individual subdirectory, on a per-extension basis. Some
of them require manual changes to the default ELK configuration.

## JVM tuning

### How to specify the amount of memory used by a service

By default, both Elasticsearch and Logstash start with [1/4 of the total host
memory](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/parallel.html#default_heap_size) allocated to
the JVM Heap Size.

The startup scripts for Elasticsearch and Logstash can append extra JVM options from the value of an environment
variable, allowing the user to adjust the amount of memory that can be used by each component:

| Service       | Environment variable |
|---------------|----------------------|
| Elasticsearch | ES_JAVA_OPTS         |
| Logstash      | LS_JAVA_OPTS         |

To accomodate environments where memory is scarce (Docker for Mac has only 2 GB available by default), the Heap Size
allocation is capped by default to 256MB per service in the `docker-compose.yml` file. If you want to override the
default JVM configuration, edit the matching environment variable(s) in the `docker-compose.yml` file.

For example, to increase the maximum JVM Heap Size for Logstash:

```yml
logstash:

  environment:
    LS_JAVA_OPTS: -Xmx1g -Xms1g
```

### How to enable a remote JMX connection to a service

As for the Java Heap memory (see above), you can specify JVM options to enable JMX and map the JMX port on the Docker
host.

Update the `{ES,LS}_JAVA_OPTS` environment variable with the following content (I've mapped the JMX service on the port
18080, you can change that). Do not forget to update the `-Djava.rmi.server.hostname` option with the IP address of your
Docker host (replace **DOCKER_HOST_IP**):

```yml
logstash:

  environment:
    LS_JAVA_OPTS: -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.port=18080 -Dcom.sun.management.jmxremote.rmi.port=18080 -Djava.rmi.server.hostname=DOCKER_HOST_IP -Dcom.sun.management.jmxremote.local.only=false
```

## Going further

### Plugins and integrations

See the following Wiki pages:

* [External applications](https://github.com/deviantony/docker-elk/wiki/External-applications)
* [Popular integrations](https://github.com/deviantony/docker-elk/wiki/Popular-integrations)

### Swarm mode

Experimental support for Docker [Swarm mode][swarm-mode] is provided in the form of a `docker-stack.yml` file, which can
be deployed in an existing Swarm cluster using the following command:

```console
$ docker stack deploy -c docker-stack.yml elk
```

If all components get deployed without any error, the following command will show 3 running services:

```console
$ docker stack services elk
```

*:information_source: To scale Elasticsearch in Swarm mode, configure seed hosts with the DNS name `tasks.elasticsearch`
instead of `elasticsearch`.*

[elk-stack]: https://www.elastic.co/what-is/elk-stack
[xpack]: https://www.elastic.co/what-is/open-x-pack
[paid-features]: https://www.elastic.co/subscriptions
[trial-license]: https://www.elastic.co/guide/en/elasticsearch/reference/current/license-settings.html

[elastdocker]: https://github.com/sherifabdlnaby/elastdocker

[linux-postinstall]: https://docs.docker.com/install/linux/linux-postinstall/

[booststap-checks]: https://www.elastic.co/guide/en/elasticsearch/reference/current/bootstrap-checks.html
[es-sys-config]: https://www.elastic.co/guide/en/elasticsearch/reference/current/system-config.html

[win-shareddrives]: https://docs.docker.com/docker-for-windows/#shared-drives
[mac-mounts]: https://docs.docker.com/docker-for-mac/osxfs/

[builtin-users]: https://www.elastic.co/guide/en/elasticsearch/reference/current/built-in-users.html
[ls-security]: https://www.elastic.co/guide/en/logstash/current/ls-security.html
[sec-tutorial]: https://www.elastic.co/guide/en/elasticsearch/reference/current/security-getting-started.html

[connect-kibana]: https://www.elastic.co/guide/en/kibana/current/connect-to-elasticsearch.html
[index-pattern]: https://www.elastic.co/guide/en/kibana/current/index-patterns.html

[config-es]: ./elasticsearch/config/elasticsearch.yml
[config-kbn]: ./kibana/config/kibana.yml
[config-ls]: ./logstash/config/logstash.yml

[es-docker]: https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html
[kbn-docker]: https://www.elastic.co/guide/en/kibana/current/docker.html
[ls-docker]: https://www.elastic.co/guide/en/logstash/current/docker-config.html

[log4j-props]: https://github.com/elastic/logstash/tree/7.6/docker/data/logstash/config
[esuser]: https://github.com/elastic/elasticsearch/blob/7.6/distribution/docker/src/docker/Dockerfile#L23-L24

[upgrade]: https://www.elastic.co/guide/en/elasticsearch/reference/current/setup-upgrade.html

[swarm-mode]: https://docs.docker.com/engine/swarm/
