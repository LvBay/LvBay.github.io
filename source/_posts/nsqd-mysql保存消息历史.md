---
title: nsqd+mysql保存消息历史
date: 2018-02-26 15:33:48
tags: nsq
---
{% asset_img nsqd.png nsqd消息分发流程%}

## 改造原因
nsqd自带消息持久化特性（nsqd的消费者因为某些原因断掉，在重新连接后仍然可以继续消费）。团队使用nsqd作为消息队列时，遇到一些问题：虽然nsqd自带消息持久化特性（nsqd的消费者因为某些原因断掉，在重新连接后仍然可以继续消费），也可以通过nsqadmin管理页面查看nsq各个节点的运行情况，但无法了解具体某个消息的生命周期。如果能将nsqd的推送记录和消费记录保存在mysql中，则可以解决痛点。

## 尝试解决方案
### 方案1：对go-nsq进行简单封装

- 在gopath下,新建znsq目录,作为项目包
{% asset_img znsq_dir.png znsq项目目录%}

- 对生产者的封装
```
package znsq

import (
	"log"
	"os"
	"znsq/models"

	"github.com/astaxie/beego/orm"
	"github.com/nsqio/go-nsq"
)

type Producer struct {
	P *nsq.Producer
}

// 初始化生产者
func NewProducer(addr string, mysql string) (wr *Producer) {
	var err error
	cfg := nsq.NewConfig()
	p, err := nsq.NewProducer(addr, cfg)
	if err != nil {
		panic(err)
	}
	p.SetLogger(log.New(os.Stdout, "nsq:", 0), nsq.LogLevelInfo)
	wr = &Producer{P: p}
	orm.RegisterDataBase("default", "mysql", mysql, 10, 10) // 注册数据库
	return wr
}

// 对go-nsq的Publish方法封装
func (p *Producer) Publish(topic string, body []byte) error {
	go p.PublishLog(topic, body) // 添加日志
	return p.P.Publish(topic, body)
}

func (p *Producer) PublishLog(topic string, body []byte) (int64, error) {
	log := &models.NsqPublishLog{}
	// log.MessageId = "" // 因为messageId由nsqd生成,所以这里还无法获取messageId
	log.Message = string(body)
	log.NsqdUrl = p.P.String()
	log.Topic = topic
	return models.AddNsqPublishLog(log)
}
```

- 对消费者的封装
```
package znsq

import (
	"fmt"
	"time"
	"znsq/models"

	"github.com/astaxie/beego/orm"
	_ "github.com/go-sql-driver/mysql"
	nsq "github.com/nsqio/go-nsq"
)

func NewNsqConsumer(topic, channel, address, mysql string, handle nsq.Handler) *nsq.Consumer {
	cfg := nsq.NewConfig()
	cfg.LookupdPollInterval = time.Second
	c, err := nsq.NewConsumer(topic, channel, cfg) // 新建一个消费者
	if err != nil {
		panic(err)
	}
	c.SetLogger(nil, 0)

	rg := &handlerRegist{
		h:       handle,
		topic:   topic,
		channel: channel,
		nsqd:    address,
	}
	c.AddHandler(rg)                                        // 添加消费者接口
	orm.RegisterDataBase("default", "mysql", mysql, 10, 10) // 注册数据库
	if err := c.ConnectToNSQD(address); err != nil {
		panic(err)
	}
	return c
}

type handlerRegist struct {
	h       nsq.Handler
	topic   string
	channel string
	nsqd    string
}

// 对调用方的handel封装
func (rg *handlerRegist) HandleMessage(message *nsq.Message) error {
	go rg.ConsumeLog(message) // 不知道会不会造成gc压力
	return rg.h.HandleMessage(message)
}

// 添加消费日志到mysql
func (rg *handlerRegist) ConsumeLog(message *nsq.Message) (int64, error) {
	log := &models.NsqConsumeLog{}
	log.NsqdUrl = rg.nsqd
	log.Topic = rg.topic
	log.Channel = rg.channel
	log.Message = string(message.Body)
	log.MessageId = fmt.Sprintf("%s", message.ID)
	return models.AddNsqConsumeLog(log)
}
```

该方法的优点：在不对官方项目做任何修改的前提下，就可以记录消息生命周期。
该方法的缺点：因为nsqd的publish消息成功的时候，不会返回message_id,而且go-nsq项目的publish方法的返回值只有一个error类型，所以在记录publish日志时，无法获得message_id，这样就无法查看某一个消息的publish和consume的日志。

### 方案2：修改go-nsq代码以及nsqd代码

- 根据文章中的图片找到publish和consum相关部分的代码

- 修改go-nsq的publish方法，额外添加一个返回值
```
// 没有亲自实现,代码略
```

- 修改nsqd项目的protocal.Pub方法，额外返回一个message_id
```
// github.com/nsqio/nsq/protocal_v2.go
func (p *protocolV2) PUB(client *clientV2, params [][]byte) ([]byte, error) {
	// ...
	return []byte(fmt.Sprintf("%s %s", okBytes, msg.ID)), nil
}
```

该方法的优点：解决了方案1的问题。
该方法的缺点：需要修改2个官方项目，改动过大。

### 方案3：只修改nsqd代码，为nsqd运行选项添加 -mysql参数

- 添加publish_log
```
// github.com/LvBay/nsq/nsqd/topic.go
func (t *Topic) put(m *Message) error {
	select {
	case t.memoryMsgChan <- m:
	default:
		b := bufferPoolGet()
		err := writeMessageToBackend(b, m, t.backend)
		bufferPoolPut(b)
		t.ctx.nsqd.SetHealth(err)
		if err != nil {
			t.ctx.nsqd.logf(LOG_ERROR,
				"TOPIC(%s) ERROR: failed to write message to backend - %s",
				t.name, err)
			return err
		}
	}
	if t.ctx.nsqd.getOpts().MysqlUrl != "" {
		go t.PublishLog(m)
	}
	return nil
}

// github.com/LvBay/nsq/nsqd/topic.go
func (t *Topic) PublishLog(m *Message) error {
	log := &NsqPublishLog{}
	log.Topic = t.name
	log.Message = string(m.Body)
	log.NsqdUrl = t.ctx.nsqd.getOpts().TCPAddress
	log.MessageId = fmt.Sprintf("%s", m.ID)
	_, err := AddNsqPublishLog(log)
	if err != nil {
		beego.Error(err)
	}
	return err
}
```

- 添加consume_log
```
// github.com/LvBay/nsq/nsqd/protocol_v2.go
func (p *protocolV2) messagePump(client *clientV2, startedChan chan bool) {
	// ...

	for {
		// ...

		select {
		// ...
		case msg := <-memoryMsgChan:
			if sampleRate > 0 && rand.Int31n(100) > sampleRate {
				continue
			}
			msg.Attempts++

			subChannel.StartInFlightTimeout(msg, client.ID, msgTimeout)
			client.SendingMessage()
			err = p.SendMessage(client, msg, &buf)
			if err != nil {
				goto exit
			}
			// 添加consume日志
			if client.ctx.nsqd.getOpts().MysqlUrl != "" {
				go client.Channel.ConsumeLog(msg)
			}
			flushed = false
		case <-client.ExitChan:
			goto exit
		}
	}

exit:
	p.ctx.nsqd.logf(LOG_INFO, "PROTOCOL(V2): [%s] exiting messagePump", client)
	heartbeatTicker.Stop()
	outputBufferTicker.Stop()
	if err != nil {
		p.ctx.nsqd.logf(LOG_ERROR, "PROTOCOL(V2): [%s] messagePump error - %s", client, err)
	}
}

// github.com/LvBay/nsq/nsqd/channel.go
func (c *Channel) ConsumeLog(m *Message) error {
	log := &NsqConsumeLog{}
	log.Topic = c.topicName
	log.Channel = c.name
	log.Message = string(m.Body)
	log.NsqdUrl = c.ctx.nsqd.getOpts().TCPAddress
	log.MessageId = fmt.Sprintf("%s", m.ID)
	_, err := AddNsqConsumeLog(log)
	if err != nil {
		beego.Error(err)
	}
	return err
}
```

- 添加 -mysql参数
```
// github.com/LvBay/nsq/app/nsqd/nsqd.go
func nsqdFlagSet(opts *nsqd.Options) *flag.FlagSet {
	// ...
	// mysql
	flagSet.String("mysql", opts.MysqlUrl, "save messages in mysql")
	return flagSet
}
```

该方法的优点：解决了方案1中message_id的问题，同时也只修改了nsqd组件。
该方法的缺点：因为需要与mysql交互，在nsqd项目中加入了beego的orm模块代码，对nsqd项目入侵较严重。

## 总结
我们团队最终采用了第三种方案,接下来准备配合前端同学修改nsqdadmin组件,将消息历史展示在管理页面