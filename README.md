# HA Singleton Service - JBoss EAP 6.4

The following procedure demonstrates how to implement a ha-singleton-service that is wrapped with the SingletonService decorator and used as a cluster-wide singleton service. The service activates a scheduled timer, which is started only once in the cluster.

## Write the HA singleton service application

The following is a simple example of a Service that is wrapped with the SingletonService decorator to be deployed as a singleton service.

### Create a service

A step by step series of examples that tell you how to get a development env running

Say what the step will be

```java
package co.kodika.jboss.singleton.service;

import java.util.Date;
import java.util.concurrent.atomic.AtomicBoolean;

import javax.naming.InitialContext;
import javax.naming.NamingException;

import org.jboss.logging.Logger;
import org.jboss.msc.service.Service;
import org.jboss.msc.service.ServiceName;
import org.jboss.msc.service.StartContext;
import org.jboss.msc.service.StartException;
import org.jboss.msc.service.StopContext;


public class HATimerService implements Service<String> {
    private static final Logger LOGGER = Logger.getLogger(HATimerService.class);
    public static final ServiceName SINGLETON_SERVICE_NAME = ServiceName.JBOSS.append("kodika", "ha", "singleton", "timer");

    /**
     * A flag whether the service is started.
     */
    private final AtomicBoolean started = new AtomicBoolean(false);

    /**
     * @return the name of the server node
     */
    public String getValue() throws IllegalStateException, IllegalArgumentException {
        LOGGER.infof("%s is %s at %s", HATimerService.class.getSimpleName(), (started.get() ? "started" : "not started"), System.getProperty("jboss.node.name"));
        return "";
    }

    public void start(StartContext arg0) throws StartException {
        if (!started.compareAndSet(false, true)) {
            throw new StartException("The service is still started!");
        }
        LOGGER.info("Start HASingleton timer service '" + this.getClass().getName() + "'");

        final String node = System.getProperty("jboss.node.name");
        try {
            InitialContext ic = new InitialContext();
            ((Scheduler) ic.lookup("global/jboss-cluster-ha-singleton-service/SchedulerBean!org.jboss.as.quickstarts.cluster.hasingleton.service.ejb.Scheduler")).initialize("HASingleton timer @" + node + " " + new Date());
        } catch (NamingException e) {
            throw new StartException("Could not initialize timer", e);
        }
    }

    public void stop(StopContext arg0) {
        if (!started.compareAndSet(true, false)) {
            LOGGER.warn("The service '" + this.getClass().getName() + "' is not active!");
        } else {
            LOGGER.info("Stop HASingleton timer service '" + this.getClass().getName() + "'");
            try {
                InitialContext ic = new InitialContext();
                ((Scheduler) ic.lookup("global/jboss-cluster-ha-singleton-service/SchedulerBean!org.jboss.as.quickstarts.cluster.hasingleton.service.ejb.Scheduler")).stop();
            } catch (NamingException e) {
                LOGGER.error("Could not stop timer", e);
            }
        }
    }
}
```

## Create an activator that installs the Service as a clustered singleton

```java
package co.kodika.jboss.singleton.service;

import org.jboss.as.clustering.singleton.SingletonService;
import org.jboss.logging.Logger;
import org.jboss.msc.service.DelegatingServiceContainer;
import org.jboss.msc.service.ServiceActivator;
import org.jboss.msc.service.ServiceActivatorContext;
import org.jboss.msc.service.ServiceController;


/**
 * Service activator that installs the HATimerService as a clustered singleton service
 * during deployment.
 *
 */
public class HATimerServiceActivator implements ServiceActivator {
    private final Logger log = Logger.getLogger(this.getClass());

    @Override
    public void activate(ServiceActivatorContext context) {
        log.info("HATimerService will be installed!");

        HATimerService service = new HATimerService();
        SingletonService<String> singleton = new SingletonService<String>(service, HATimerService.SINGLETON_SERVICE_NAME);
        /*
         * To pass a chain of election policies to the singleton, for example, 
         * to tell JGroups to prefer running the singleton on a node with a
         * particular name, uncomment the following line:
         */
        // singleton.setElectionPolicy(new PreferredSingletonElectionPolicy(new SimpleSingletonElectionPolicy(), new NamePreference("node1/singleton")));

        singleton.build(new DelegatingServiceContainer(context.getServiceTarget(), context.getServiceRegistry()))
                .setInitialMode(ServiceController.Mode.ACTIVE)
                .install()
        ;
    }
}
```

### Create a ServiceActivator File

Create a file named org.jboss.msc.service.ServiceActivator in the application's resources/META-INF/services/ directory. Add a line containing the fully qualified name of the ServiceActivator class created in the previous step.

```
co.kodika.jboss.singleton.service.HATimerServiceActivator
```

### Create a Singleton bean that implements a timer to be used as a cluster-wide singleton timer

This Singleton bean must not have a remote interface and you must not reference its local interface from another EJB in any application. This prevents a lookup by a client or other component and ensures the SingletonService has total control of the Singleton.

#### Create a Interface

```java
package co.kodika.jboss.singleton.service;

public interface Scheduler {

    void initialize(String info);

    void stop();

}
```

## Create the Singleton bean that implements the cluster-wide singleton timer

```java
package co.kodika.jboss.singleton.service;

import javax.annotation.Resource;
import javax.ejb.ScheduleExpression;
import javax.ejb.Singleton;
import javax.ejb.Timeout;
import javax.ejb.Timer;
import javax.ejb.TimerConfig;
import javax.ejb.TimerService;

import org.jboss.logging.Logger;


/**
 * A simple example to demonstrate a implementation of a cluster-wide singleton timer.
 *
 */
@Singleton
public class SchedulerBean implements Scheduler {
    private static Logger LOGGER = Logger.getLogger(SchedulerBean.class);
    @Resource
    private TimerService timerService;

    @Timeout
    public void scheduler(Timer timer) {
        LOGGER.info("HASingletonTimer: Info=" + timer.getInfo());
    }

    @Override
    public void initialize(String info) {
        ScheduleExpression sexpr = new ScheduleExpression();
        // set schedule to every 10 seconds for demonstration
        sexpr.hour("*").minute("*").second("0/10");
        // persistent must be false because the timer is started by the HASingleton service
        timerService.createCalendarTimer(sexpr, new TimerConfig(info, false));
    }

    @Override
    public void stop() {
        LOGGER.info("Stop all existing HASingleton timers");
        for (Timer timer : timerService.getTimers()) {
            LOGGER.trace("Stop HASingleton timer: " + timer.getInfo());
            timer.cancel();
        }
    }
}
```

## Sources

* [JBoss Red Hat](https://access.redhat.com/documentation/) - JBoss EAP 6.4
