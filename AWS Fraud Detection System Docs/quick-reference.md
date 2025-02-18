## Component Quick Reference

### Ingestion
```
- API: REST, $1/million calls
- Lambda: 128MB, 10s timeout
- Kinesis: On-demand, auto-scaling
```

### Storage
```
- Standard: 0-30 days ($0.023/GB)
- IA: 30-90 days ($0.0125/GB)
- Intelligent: 90+ days (auto-optimize)
- Retention: 365 days max
```

### ML Infrastructure
```
- Instance: t2.medium ($0.042/hour)
- Memory: 4GB
- Throughput: 100 TPS
- Latency: ~100ms
```

### Alerts
```
- DynamoDB: TTL 90 days
- SNS: Multi-channel
- Channels: Email/SMS/App
- Latency: <5s
```

## Performance Summary
```
Throughput: 1000 TPS
Storage: Auto-tiered
Cost: ~$100/month @ 100k tx
Availability: 99.95%
```
