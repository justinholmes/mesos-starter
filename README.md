Spring Boot starter for Mesos
===

Spring Boot starter package for writing Mesos frameworks

Features
---
- Vertical scaling
- Deploy executor on all slaves
- Support for Docker containers

Getting Started
---
Start by adding the `Spring-boot-mesos-starter` dependency to your project

```
<dependency>
    <groupId>dk.mwl.mesos</groupId>
    <artifactId>spring-boot-mesos-starter</artifactId>
    <version>0.1</version>
</dependency>
```

Make sure your Spring Boot application has a name, by adding the `spring.application.name` to the `application.properties` file i.e.

```
spring.application.name=Sample application
```

To run three instances of a Docker image add the following in `application.properties`

```
mesos.resources.distinctSlave=true
mesos.resources.scale=3
mesos.resources.cpus=0.1
mesos.resources.mem=64
mesos.docker.mage=tutum/hello-world:latest
```

That is all you need to do if you have annotated your Spring configuration with `@SpringBootApplication` or `@EnableAutoConfiguration`.

```
@SpringBootApplication
public class SampleApplication {

  public static void main(String[] args) {
    SpringApplication.run(SampleApplication.class, args);
  }
}
```

For a complete example, see `mesos-starter-sample` module.

Starting the application
---

The only required parameter for the scheduler is `mesos.master`. The value of the parameter is passed directly to the Mesos Scheduler Driver which allows the following formats

- `host:port`
- `zk://host1:port1,host2:port2,.../path`
- `zk://username:password@host1:port1,host2:port2,.../path`
- `file:///path/to/file`

For resiliency it is also recommended to point the application to a Zookeeper cluster and giving the instance a name.
```
mesos.zookeeper.server=zookeeper:2181
mesos.framework.name=sampleApp1
```

The purpose of `mesos.framework.name` is to distinguish instances of the scheduler.

Offers evaluation
===

Mesos-starter offers a set of offer evaluation rules
- Physical requirements
- Distinct slave
- Scale factor
- Role assigned

They all work in combination with each other, though this might change in the future.

Physical requirements
---
Reject offers that does not have the required amount of either CPUs, memory or ports.
At his time the `ports` property is an amount and ports are dynamically assigned. Work is being done on a rule for requiring specific ports, including privileged ports.

Distinct slave
---
This rule will make sure that offers for hosts where the application is already running are being rejected.

Scale factor
---
This rule will make sure that only a certain number of instances are running in the Mesos cluster. It is currently not possible to change the scale factor at runtime, but it's very high on the backlog. Furthermore it'll also be possible to insert your own scale factor bean.
It is recommended to have this rule enabled in most cases.

Role assigned
---
This rule only accept offers assigned to the Role defined in `mesos.role`.

Use cases
---
A few good examples

### Stateless web application
For a stateless web application that can run anywhere in the cluster with only a requirement for a single network port, the following should be sufficient

```
mesos.resources.scale=3
mesos.resources.cpus=0.1
mesos.resources.mem=64
mesos.resources.ports=1
```

This will run 3 instances of the application with one port exposed. Bare in mind that they all might run on the very same host.

### Distributed database application
For a distributed database you want to run a certain number of instances and never more than one on every host. To achieve that you can enable `scale` and `distinctSlave`, like

```
mesos.resources.scale=3
mesos.resources.distinctSlave=true
mesos.resources.cpus=0.1
mesos.resources.mem=64
mesos.resources.ports=1
```

### Cluster wide system demon
Often operations would like to run a single application on each host in the cluster to harvest information from every single node. This can be achieved by not adding the Scale factor rule and adding the Distinct slave rule.

```
mesos.resources.distinctSlave=true
mesos.resources.cpus=0.1
mesos.resources.mem=64
```

Another, safer, way to achieve the same result is by assigning resources to a specific role on all nodes. I.e. by adding the following to `/etc/mesos-slave/resources`
```
cpus(myDemon):0.2; mem(myDemon):64; ports(myDemon):[514-514];
```

And configure the scheduler with the following options

```
mesos.role=myDemon
mesos.resources.distinctSlave=true
mesos.resources.role=all
```

This way the scheduler will take all resources allocated to the role and make sure it's only running once in every single slave.

It is always recommended to run the scheduler with a Mesos role and reserved resources in such cases to make sure that scheduler is being offered resources for all nodes in the cluster.

Framework shutdown
===

The scheduler can survive `SIGKILL` or being lost (system crashes etc.). If you want to completely de-register the framework and shutdown all tasks, just stop the scheduler with a plain `SIGTERM`.
