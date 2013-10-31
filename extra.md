# Extra字段定义

### Sender

消息(请求或回复)的发起者，发起者如果具有唯一的标识，可以在这个字段记录。

中间节点，例如carabiner不应该改写Sender字段。

### Via

消息传递时经过的中间节点(例如carabiner)，在此记录中间节点的标识与时间。每个中间节点在extra frames最后追加一行记录。`Via: id, timestamp`，例如:

```python
msgpack(('Via', carabiner_id, time.time()))
```

### Version

- 当消息是来自客户端的请求时，这个字段表示要求的版本。如果没有发送版本要求，一般当作对稳定版本的请求。(版本的规则请参考Kite-Line)
- 当消息来自服务端的回复时，这个字段表示处理这个请求的服务的实际版本。


