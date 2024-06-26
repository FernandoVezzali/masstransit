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

## Sending Message to a Dead-Letter Topic

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

## Invalid messages

When a consumer determines that a message cannot be processed because it is invalid, the best action is to ensure that the message is not retried, avoiding unnecessary processing and potential endless loops. In such cases, you can implement the following steps:

- Move the message to an error queue or topic: This approach ensures that the invalid message is logged and stored separately for further analysis or manual intervention.
- Log the error: Record detailed information about the invalid message and the reason it could not be processed, which helps in diagnosing and resolving the issue.
- Acknowledge the message: Acknowledge the message to avoid it being retried by the consumer.

Here’s how you can implement these steps in a MassTransit consumer:

Step 1: Define the Error Queue or Topic
First, configure an error queue or topic where invalid messages will be sent.

```
public class InvalidMessageConsumer : IConsumer<YourMessage>
{
    public async Task Consume(ConsumeContext<YourMessage> context)
    {
        try
        {
            // Validate the message
            if (!IsValid(context.Message))
            {
                throw new InvalidMessageException("Invalid message");
            }

            // Your message processing logic here
        }
        catch (InvalidMessageException ex)
        {
            // Log the invalid message
            LogInvalidMessage(context.Message, ex);

            // Move the message to the error queue
            await context.Send<YourMessage>("error-queue", context.Message);

            // Acknowledge the message
            await context.ConsumeCompleted;
        }
        catch (Exception ex)
        {
            // Handle other exceptions (e.g., retries, redelivery)
            throw;
        }
    }

    private bool IsValid(YourMessage message)
    {
        // Implement your validation logic here
        return true; // Placeholder
    }

    private void LogInvalidMessage(YourMessage message, Exception ex)
    {
        // Implement your logging logic here
        Console.WriteLine($"Invalid message: {message}, Error: {ex.Message}");
    }
}
```

Step 2: Configure the Consumer with Error Handling
Configure your MassTransit bus to use the error queue.

```
var busControl = Bus.Factory.CreateUsingKafka(cfg =>
{
    cfg.Host("localhost:9092");

    cfg.ReceiveEndpoint("your-topic", e =>
    {
        e.Consumer<InvalidMessageConsumer>();
    });

    cfg.ReceiveEndpoint("error-queue", e =>
    {
        e.Consumer<ErrorConsumer>(); // Consumer to handle invalid messages if needed
    });
});
```

Step 3: Implement the ErrorConsumer (Optional)
If you want to process or log messages in the error queue, you can implement an ErrorConsumer.

```
public class ErrorConsumer : IConsumer<YourMessage>
{
    public async Task Consume(ConsumeContext<YourMessage> context)
    {
        // Implement your logic to handle messages in the error queue
        Console.WriteLine($"Error queue received message: {context.Message}");
    }
}
```

Summary
- Validation: Validate the message and determine if it is invalid.
- Logging: Log detailed information about the invalid message.
- Error Handling: Send the invalid message to an error queue or topic for further analysis.
- Acknowledgement: Acknowledge the message to prevent retries.
 
By following these steps, you can effectively handle invalid messages and ensure they are not retried, while still retaining the ability to analyze and resolve the issues causing them to be invalid.

## Poison Message VS Invalid Message

An invalid message can be considered a poison message, but they are not always the same thing. Here's the distinction:

### Poison Message
A poison message is a message that cannot be processed by a consumer due to some issue, often resulting in repeated failures. Poison messages typically cause the consumer to crash or retry indefinitely, which can disrupt normal processing. Common reasons for poison messages include:

- Corrupt data
- Format errors
- Logic errors in the consumer

### Invalid Message
An invalid message is one that fails validation based on business rules or schema requirements. It is a broader term that includes any message deemed not suitable for processing. Reasons for invalid messages can include:

- Missing required fields
- Data values out of acceptable ranges
- Violation of business rules

## Naming conventions

Naming conventions for dead-letter topics and error queues should be clear and consistent, making it easy to understand their purpose. For a regular topic named add-transaction, here are some examples:

### Dead-Letter Topic
A dead-letter topic is typically used for messages that cannot be processed and need to be revisited or analyzed later.

- add-transaction-dlt
- add-transaction-dead-letter
- add-transaction-dlq (dead-letter queue)
- add-transaction-failed

### Error Queue
An error queue is used to capture messages that fail validation or processing due to errors.

- add-transaction-error
- add-transaction-error-queue
- add-transaction-err
- add-transaction-invalid
