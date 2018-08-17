# DEMO
```go
// 创建consumer
consumer, err := nsq.NewConsumer(topics[i], *channel, cfg)
if err != nil {
    log.Fatal(err)
}

// 绑定消费到数据后的回调，绑定同时会启动线程监听incomingMessages
consumer.AddConcurrentHandlers(&TailHandler{},1)

// 连接某个节点
err = consumer.ConnectToNSQDs(nsqdTCPAddrs)
if err != nil {
    log.Fatal(err)
}
// 或者连接lookupd
err = consumer.ConnectToNSQLookupds(lookupdHTTPAddrs)
if err != nil {
    log.Fatal(err)
}

```


`TailHandler`类要实现一个`HandleMessage`方法
```go
func (th *TailHandler) HandleMessage(m *nsq.Message) error {
	_, err := os.Stdout.Write(m.Body)
	if err != nil {
		log.Fatalf("ERROR: failed to write to os.Stdout - %s", err)
	}
	_, err = os.Stdout.WriteString("\n")
	if err != nil {
		log.Fatalf("ERROR: failed to write to os.Stdout - %s", err)
	}
	
	return nil
}
```


# 分析
cusumer的监听实现是写在`github.com/nsqio/nsq/vendor/github.com/nsqio/go-nsq`包下

```go
func (r *Consumer) AddConcurrentHandlers(handler Handler, concurrency int) {
	if atomic.LoadInt32(&r.connectedFlag) == 1 {
		panic("already connected")
	}

	atomic.AddInt32(&r.runningHandlers, int32(concurrency))
	for i := 0; i < concurrency; i++ {
		go r.handlerLoop(handler)
	}
}

func (r *Consumer) handlerLoop(handler Handler) {
	r.log(LogLevelDebug, "starting Handler")

	for {
		message, ok := <-r.incomingMessages
		if !ok {
			goto exit
		}
        // 消息的最大尝试次数超出配置，如果consumer有定义FailedMessageLogger类型的handler，就调用，没有就跳过
		if r.shouldFailMessage(message, handler) {
			message.Finish()
			continue
		}
        // 传递消息
		err := handler.HandleMessage(message)
		if err != nil {
			r.log(LogLevelError, "Handler returned error (%s) for msg %s", err, message.ID)
			if !message.IsAutoResponseDisabled() {
				message.Requeue(-1)
			}
			continue
		}
        // 如果没有关闭,就响应autoResponseDisabled
		if !message.IsAutoResponseDisabled() {
			message.Finish()
		}
	}

exit:
	r.log(LogLevelDebug, "stopping Handler")
	if atomic.AddInt32(&r.runningHandlers, -1) == 0 {
		r.exit()
	}
}
```


连接节点(lookup)
```go
// if this is the first one, kick off the go loop
if numLookupd == 1 {
    // 请求lookup服务查询topic所在的节点信息，并调用ConnectToNSQD尝试直连
    r.queryLookupd()
    r.wg.Add(1)
    go r.lookupdLoop()
}

func (r *Consumer) ConnectToNSQD(addr string) error {
	if atomic.LoadInt32(&r.stopFlag) == 1 {
		return errors.New("consumer stopped")
	}

	if atomic.LoadInt32(&r.runningHandlers) == 0 {
		return errors.New("no handlers")
	}

	atomic.StoreInt32(&r.connectedFlag, 1)

	logger, logLvl := r.getLogger()

        // 建立连接对象
	conn := NewConn(addr, &r.config, &consumerConnDelegate{r})
	conn.SetLogger(logger, logLvl,
		fmt.Sprintf("%3d [%s/%s] (%%s)", r.id, r.topic, r.channel))

    r.mtx.Lock()
        // 先放入pendingConnections
	_, pendingOk := r.pendingConnections[addr]
	_, ok := r.connections[addr]
	if ok || pendingOk {
		r.mtx.Unlock()
		return ErrAlreadyConnected
	}
	r.pendingConnections[addr] = conn
	if idx := indexOf(addr, r.nsqdTCPAddrs); idx == -1 {
		r.nsqdTCPAddrs = append(r.nsqdTCPAddrs, addr)
	}
	r.mtx.Unlock()

	r.log(LogLevelInfo, "(%s) connecting to nsqd", addr)

	cleanupConnection := func() {
		r.mtx.Lock()
		delete(r.pendingConnections, addr)
		r.mtx.Unlock()
		conn.Close()
	}
        // 连接tcp服务端口，同时启动read和write两个线程
	resp, err := conn.Connect()
	if err != nil {
		cleanupConnection()
		return err
	}

	if resp != nil {
		if resp.MaxRdyCount < int64(r.getMaxInFlight()) {
			r.log(LogLevelWarning,
				"(%s) max RDY count %d < consumer max in flight %d, truncation possible",
				conn.String(), resp.MaxRdyCount, r.getMaxInFlight())
		}
	}

        // 发送一个sub命令给节点
	cmd := Subscribe(r.topic, r.channel)
	err = conn.WriteCommand(cmd)
	if err != nil {
		cleanupConnection()
		return fmt.Errorf("[%s] failed to subscribe to %s:%s - %s",
			conn, r.topic, r.channel, err.Error())
	}

        // 连接对象从pendingConnections移入connections
	r.mtx.Lock()
	delete(r.pendingConnections, addr)
	r.connections[addr] = conn
	r.mtx.Unlock()

	// pre-emptive signal to existing connections to lower their RDY count
	for _, c := range r.conns() {
		r.maybeUpdateRDY(c)
	}

	return nil
}

func (c *Conn) readLoop() {
	delegate := &connMessageDelegate{c}
	for {
		if atomic.LoadInt32(&c.closeFlag) == 1 {
			goto exit
		}
        // 循环阻塞的从连接上读取数据，根据类型判断处理
		frameType, data, err := ReadUnpackedResponse(c)
		if err != nil {
			if !strings.Contains(err.Error(), "use of closed network connection") {
				c.log(LogLevelError, "IO error - %s", err)
				c.delegate.OnIOError(c, err)
			}
			goto exit
		}

        // 心跳
		if frameType == FrameTypeResponse && bytes.Equal(data, []byte("_heartbeat_")) {
			c.log(LogLevelDebug, "heartbeat received")
			c.delegate.OnHeartbeat(c)
			err := c.WriteCommand(Nop())
			if err != nil {
				c.log(LogLevelError, "IO error - %s", err)
				c.delegate.OnIOError(c, err)
				goto exit
			}
			continue
		}

		switch frameType {
		case FrameTypeResponse:
			c.delegate.OnResponse(c, data)
        case FrameTypeMessage:
            // 实际需要的消息数据，先解码，最终传递给incomingMessages
			msg, err := DecodeMessage(data)
			if err != nil {
				c.log(LogLevelError, "IO error - %s", err)
				c.delegate.OnIOError(c, err)
				goto exit
			}
			msg.Delegate = delegate
			msg.NSQDAddress = c.String()

			atomic.AddInt64(&c.rdyCount, -1)
			atomic.AddInt64(&c.messagesInFlight, 1)
			atomic.StoreInt64(&c.lastMsgTimestamp, time.Now().UnixNano())

			c.delegate.OnMessage(c, msg)
		case FrameTypeError:
			c.log(LogLevelError, "protocol error - %s", data)
			c.delegate.OnError(c, data)
		default:
			c.log(LogLevelError, "IO error - %s", err)
			c.delegate.OnIOError(c, fmt.Errorf("unknown frame type %d", frameType))
		}
	}

exit:
	atomic.StoreInt32(&c.readLoopRunning, 0)
	// start the connection close
	messagesInFlight := atomic.LoadInt64(&c.messagesInFlight)
	if messagesInFlight == 0 {
		// if we exited readLoop with no messages in flight
		// we need to explicitly trigger the close because
		// writeLoop won't
		c.close()
	} else {
		c.log(LogLevelWarning, "delaying close, %d outstanding messages", messagesInFlight)
	}
	c.wg.Done()
	c.log(LogLevelInfo, "readLoop exiting")
}

func (c *Conn) writeLoop() {
	for {
		select {
        case <-c.exitChan:
                        // 关闭
			c.log(LogLevelInfo, "breaking out of writeLoop")
			// Indicate drainReady because we will not pull any more off msgResponseChan
			close(c.drainReady)
			goto exit
        case cmd := <-c.cmdChan:
                        // 发送命令
			err := c.WriteCommand(cmd)
			if err != nil {
				c.log(LogLevelError, "error sending command %s - %s", cmd, err)
				c.close()
				continue
			}
		case resp := <-c.msgResponseChan:
			// Decrement this here so it is correct even if we can't respond to nsqd
			msgsInFlight := atomic.AddInt64(&c.messagesInFlight, -1)

			if resp.success {
				c.log(LogLevelDebug, "FIN %s", resp.msg.ID)
				c.delegate.OnMessageFinished(c, resp.msg)
				c.delegate.OnResume(c)
			} else {
				c.log(LogLevelDebug, "REQ %s", resp.msg.ID)
				c.delegate.OnMessageRequeued(c, resp.msg)
				if resp.backoff {
					c.delegate.OnBackoff(c)
				} else {
					c.delegate.OnContinue(c)
				}
			}

			err := c.WriteCommand(resp.cmd)
			if err != nil {
				c.log(LogLevelError, "error sending command %s - %s", resp.cmd, err)
				c.close()
				continue
			}

			if msgsInFlight == 0 &&
				atomic.LoadInt32(&c.closeFlag) == 1 {
				c.close()
				continue
			}
		}
	}

exit:
	c.wg.Done()
	c.log(LogLevelInfo, "writeLoop exiting")
}
```

nsqd服务收到sub命令后怎么处理？
```go
func (p *protocolV2) SUB(client *clientV2, params [][]byte) ([]byte, error) {
	if atomic.LoadInt32(&client.State) != stateInit {
		return nil, protocol.NewFatalClientErr(nil, "E_INVALID", "cannot SUB in current state")
	}

	if client.HeartbeatInterval <= 0 {
		return nil, protocol.NewFatalClientErr(nil, "E_INVALID", "cannot SUB with heartbeats disabled")
	}

	if len(params) < 3 {
		return nil, protocol.NewFatalClientErr(nil, "E_INVALID", "SUB insufficient number of parameters")
	}

	topicName := string(params[1])
	if !protocol.IsValidTopicName(topicName) {
		return nil, protocol.NewFatalClientErr(nil, "E_BAD_TOPIC",
			fmt.Sprintf("SUB topic name %q is not valid", topicName))
	}

	channelName := string(params[2])
	if !protocol.IsValidChannelName(channelName) {
		return nil, protocol.NewFatalClientErr(nil, "E_BAD_CHANNEL",
			fmt.Sprintf("SUB channel name %q is not valid", channelName))
	}

	if err := p.CheckAuth(client, "SUB", topicName, channelName); err != nil {
		return nil, err
	}

	// This retry-loop is a work-around for a race condition, where the
	// last client can leave the channel between GetChannel() and AddClient().
	// Avoid adding a client to an ephemeral channel / topic which has started exiting.
	var channel *Channel
	for {
        // 获取topic和channel对象，绑定client对象，等有消息pub来的时候就会往client写，逻辑在github.com/nsqio/nsq/nsqd/protocol_v2.go的messagePump方法里
		topic := p.ctx.nsqd.GetTopic(topicName)
		channel = topic.GetChannel(channelName)
		channel.AddClient(client.ID, client)

		if (channel.ephemeral && channel.Exiting()) || (topic.ephemeral && topic.Exiting()) {
			channel.RemoveClient(client.ID)
			time.Sleep(1 * time.Millisecond)
			continue
		}
		break
	}
	atomic.StoreInt32(&client.State, stateSubscribed)
	client.Channel = channel
	// update message pump
	client.SubEventChan <- channel

	return okBytes, nil
}
```