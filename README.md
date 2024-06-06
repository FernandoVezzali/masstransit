# MassTransit

## Consumer

```
public class YourConsumer : IConsumer<YourMessage>
{
    public async Task Consume(ConsumeContext<YourMessage> context)
    {
        try
        {
            // message processing logic here
        }
        catch (Exception ex)
        {
            throw;
        }
    }
}
```

## Retry Policy

```
var busControl = Bus.Factory.CreateUsingKafka(cfg =>
{
    cfg.Host("localhost:9092");

    cfg.ReceiveEndpoint("your-topic", e =>
    {
        e.Consumer<YourConsumer>(consumerConfig =>
        {
            consumerConfig.UseMessageRetry(r => r.Interval(3, TimeSpan.FromSeconds(5)));
        });
    });

});
```

## Using a Dead-Letter Topic

```
public class YourConsumer : IConsumer<YourMessage>
{
    public async Task Consume(ConsumeContext<YourMessage> context)
    {
        try
        {
            // message processing logic here
        }
        catch (Exception ex)
        {
            await context.Send<YourMessage>("dead-letter-topic", context.Message);
        }
    }
}
```

## Consumer for Dead-Letter Topic with Redeliver

```
public class DeadLetterConsumer : IConsumer<YourMessage>
{
    public async Task Consume(ConsumeContext<YourMessage> context)
    {
        try
        {
            // message processing logic here
        }
        catch (Exception ex)
        {
            // Log the exception or handle it if needed

            // Schedule redelivery
            await context.Redeliver(TimeSpan.FromMinutes(10)); 
        }
    }
}

```

## Configuring Redelivery Policy

```
var busControl = Bus.Factory.CreateUsingKafka(cfg =>
{
    cfg.Host("localhost:9092");

    cfg.ReceiveEndpoint("dead-letter-topic", e =>
    {
        e.Consumer<DeadLetterConsumer>(consumerConfig =>
        {
            // Configure redelivery policy
            consumerConfig.UseScheduledRedelivery(r => r.Intervals(TimeSpan.FromMinutes(1), TimeSpan.FromMinutes(5), TimeSpan.FromMinutes(10)));
            consumerConfig.UseMessageRetry(r => r.Immediate(5)); // Immediate retries before redelivery
        });
    });
});
```
