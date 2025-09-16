# Koha Background Jobs System

## Overview

Koha's background jobs system allows long-running tasks to be processed asynchronously, improving user experience by preventing timeouts and allowing users to continue working while tasks complete in the background.

## Architecture

### Job Notification Methods

Koha supports two notification methods for background jobs, controlled by the `JobsNotificationMethod` system preference:

- **`STOMP`** (default): Uses message broker (RabbitMQ) for real-time job notifications
- **`polling`**: Uses database polling every 10 seconds for job discovery

### Key Components

1. **`Koha::BackgroundJob`** - Core job management class
2. **`misc/workers/background_jobs_worker.pl`** - General background job processor
3. **`misc/workers/es_indexer_daemon.pl`** - Elasticsearch indexing job processor

## System Preference: JobsNotificationMethod

```sql
('JobsNotificationMethod','STOMP','polling|STOMP','Define the preferred job worker notification method','Choice')
```

### STOMP Mode
- Requires RabbitMQ message broker
- Real-time job processing
- Better performance for high-volume environments
- Immediate job execution when enqueued

### Polling Mode
- No external dependencies
- Database-only approach
- 10-second polling interval
- Suitable for low-volume environments or when message broker is unavailable

## Worker Script Behavior

### Connection Logic

Both worker scripts follow this pattern:

```perl
my $notification_method = C4::Context->preference('JobsNotificationMethod') // 'STOMP';

my ( $conn, $error );
if ( $notification_method eq 'STOMP' ) {
    try {
        $conn = Koha::BackgroundJob->connect;
    } catch {
        $error = sprintf "Cannot connect to the message broker, the jobs will be processed anyway (%s)", $_;
    };
    $error ||= "Cannot connect to the message broker, the jobs will be processed anyway" unless $conn;
    warn $error if $error;
}
```

### Processing Logic

**With STOMP connection:**
- Subscribe to message queues
- Process jobs as messages arrive
- Use `ack`/`nack` for message acknowledgment

**Without STOMP connection (polling mode or connection failure):**
- Query database every 10 seconds for new jobs
- Process jobs in batches
- Update job status directly in database

## Common Issues and Solutions

### STOMP Connection Warnings

**Problem:** Workers show connection warnings even when `JobsNotificationMethod` is set to `'polling'`

**Root Cause:** Workers were attempting STOMP connections regardless of the system preference setting

**Solution:** Check `JobsNotificationMethod` before attempting any connection:

```perl
# ❌ Wrong - always attempts connection
my $conn = Koha::BackgroundJob->connect;

# ✅ Correct - check preference first
my $notification_method = C4::Context->preference('JobsNotificationMethod') // 'STOMP';
my $conn;
if ( $notification_method eq 'STOMP' ) {
    $conn = Koha::BackgroundJob->connect;
}
```

### Connection Error Handling

The `Koha::BackgroundJob->connect()` method can fail in two ways:

1. **Exception thrown** - Network errors, authentication failures
2. **Silent failure** - Returns `undef` for configuration issues

Both cases must be handled:

```perl
my ( $conn, $error );
if ( $notification_method eq 'STOMP' ) {
    try {
        $conn = Koha::BackgroundJob->connect;
    } catch {
        $error = sprintf "Cannot connect to the message broker, the jobs will be processed anyway (%s)", $_;
    };
    $error ||= "Cannot connect to the message broker, the jobs will be processed anyway" unless $conn;
    warn $error if $error;
}
```

## Best Practices

### For Developers

1. **Always check `JobsNotificationMethod`** before attempting STOMP operations
2. **Handle both exception and undef cases** for connection failures
3. **Ensure fallback to polling mode** when STOMP is unavailable
4. **Use consistent error handling patterns** across worker scripts

### For System Administrators

1. **Set `JobsNotificationMethod` appropriately** for your environment:
   - Use `'STOMP'` for high-volume production systems with RabbitMQ
   - Use `'polling'` for development or low-volume systems
2. **Monitor worker processes** to ensure they're running
3. **Configure RabbitMQ properly** when using STOMP mode
4. **Check logs** for connection warnings that might indicate configuration issues

### For Plugin Developers

When creating background jobs in plugins:

```perl
# Enqueue a job
my $job = Koha::BackgroundJob->new({
    type => 'plugin_my_job_type',
    data => encode_json($job_data),
    queue => 'default'  # or 'long_tasks'
})->store;

$job->enqueue;  # This respects JobsNotificationMethod automatically
```

## Configuration Examples

### RabbitMQ Configuration (koha-conf.xml)

```xml
<message_broker>
    <hostname>localhost</hostname>
    <port>61613</port>
    <username>koha</username>
    <password>password</password>
    <vhost>/koha</vhost>
</message_broker>
```

### System Preference Settings

```sql
-- For production with RabbitMQ
UPDATE systempreferences SET value = 'STOMP' WHERE variable = 'JobsNotificationMethod';

-- For development or simple setups
UPDATE systempreferences SET value = 'polling' WHERE variable = 'JobsNotificationMethod';
```

## Troubleshooting

### Common Error Messages

**"Cannot connect to the message broker, the jobs will be processed anyway"**
- Check if RabbitMQ is running
- Verify message_broker configuration in koha-conf.xml
- Consider switching to polling mode if STOMP is not needed

**Jobs not processing**
- Verify worker scripts are running
- Check job status in `background_jobs` table
- Review worker logs for errors

### Debugging Steps

1. Check system preference: `SELECT * FROM systempreferences WHERE variable = 'JobsNotificationMethod';`
2. Verify worker processes: `ps aux | grep worker`
3. Check job queue: `SELECT * FROM background_jobs WHERE status = 'new';`
4. Review logs: Check worker output and Koha logs for errors

## Performance Considerations

### STOMP Mode
- **Pros:** Real-time processing, better for high-volume
- **Cons:** Requires RabbitMQ, additional complexity

### Polling Mode  
- **Pros:** Simple, no external dependencies
- **Cons:** 10-second delay, database overhead

Choose based on your environment's needs and complexity tolerance.
