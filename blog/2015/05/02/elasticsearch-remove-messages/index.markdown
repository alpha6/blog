---
tags: graylog2, elasticsearch
title: Remove messages from Graylog2
---

If you want to remove all messages with requested pattern in message body, you have to write following line:

curl -XDELETE 'http://graylog.example:9200/graylog_*/message/_query' -d'{"query" : {"match": { "message" : "SearchPattern"}}}'

remove exact pattern:
curl -XDELETE 'http://graylog.example:9200/graylog_*/message/_query' -d'{"query" : {"term": { "message" : "ExactText"}}}'


