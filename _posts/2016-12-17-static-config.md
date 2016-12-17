---
title:  "static-config: 6-lines wrapper around typesafehub/config"
date:   2016-12-16 21:37:00
categories: [software-development]
tags: [scala, projects]
---

I have just released [static-config](https://github.com/Krever/static-config) which is very small library that allows to scrap some boilerplate when working with [typesafehub/config](https://github.com/typesafehub/config). Here you can find motivation for writing it.


### Motivation

I tend to use *typesafehub/config* pretty extensively and in most cases I use it in similarly way:  `application.conf` in resources + wrapping object for type safe accessing of config entries. Here is small example:

##### application.conf
```conf
my-app {
  app-name = "My Fancy App"
  kafka {
    brokers = ["broker1", "broker2"]
  }
  http {
  interface = "0.0.0.0"
    port = 8080
  }
}

```

##### Config.scala
```scala
object Config {
  val config = ConfigFactory.load().getConfig("my-app")
  
  val `app-name` = config.getString("app-name")
	
  object kafka {
    val config = Config.config.getConfig("kafka")
    val brokers = config.getStringList("brokers")
  }
	
  object http {
    val config = Config.config.getConfig("http")
    val interface = config.getString("interface")
    val port = config.getInt("port")
  }
}

```

As you can see, `Config` code is rather verbose. Every config entry is triplicated(application.conf, name of the variable and argument to the `getXY`. Moreover, for each entry we need to specify from which config we want to get our entry. That made me think.

### static-config
So I developed this tiny library to scrap (almost) all of this boilerplate. I has to mention it wouldn't be possible without great [lihaoyi/sourcecode](https://github.com/lihaoyi/sourcecode).

Here is above example written with *static-config*.

```scala
import com.typesafe.config.{Config, ConfigFactory}
import staticconfig._

object SConfigExample extends SConfig {

  override def config: Config = ConfigFactory.load()
  
  object `my-app` extends LocalSConfigNode {
    val `app-name` = configEntry(_.getString)
  
    object kafka extends LocalSConfigNode {
      val brokers = configEntry(_.getStringList)
    }
  
    object http extends LocalSConfigNode {
      val interface = configEntry(_.getString)
      val port = configEntry(_.getInt)
    }
  }
}
```
No boilerplate left! Success! (In fact we could make `ConfigFactory.load()` a default implementation of `config` but that has some drawbacks. Let's just act like it is not there.)

### How it works?
I didn't lie, this library has only 6 relevant lines of code. If you're interested [go and read it](https://github.com/Krever/static-config/blob/master/src/main/scala/staticconfig/SConfig.scala)! 
