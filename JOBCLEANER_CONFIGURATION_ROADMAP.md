# River JobCleaner Configuration Enhancement Roadmap

## Problem Analysis

**Current State :**
- JobCleaner runs every 30 seconds (hardcoded via `JobCleanerIntervalDefault`)
- Batch size fixed at 1,000 jobs (hardcoded via `BatchSizeDefault`)
- 10.5-second cleanup operations on ~1M job database causing high DB load in our case
- No configuration options for tuning performance

**Goal:** Add configurable interval and batch size following River's established patterns.

##  Implementation Plan

### Phase 1: Client Configuration (client.go)

Add two new fields to the existing `Config` struct following the established pattern:

```go
// In client.go, add to Config struct around line 174
type Config struct {
    // ... existing fields ...
    
    // JobCleanerInterval sets how often the job cleaner runs to remove old jobs.
    // A longer interval reduces database load but means old jobs remain longer.
    //
    // Defaults to 30 seconds.
    JobCleanerInterval time.Duration
    
    // JobCleanerBatchSize sets the maximum number of jobs to delete in a single
    // cleanup operation. Larger values are more efficient but increase database
    // transaction size and lock duration.
    //
    // Defaults to 1000.
    JobCleanerBatchSize int
}
```

### Phase 2: Wire Configuration (client.go)

Update the `WithDefaults()` method around line 404:

```go
func (c *Config) WithDefaults() *Config {
    return &Config{
        // ... existing defaults ...
        JobCleanerInterval:  cmp.Or(c.JobCleanerInterval, riversharedmaintenance.JobCleanerIntervalDefault),
        JobCleanerBatchSize: cmp.Or(c.JobCleanerBatchSize, riversharedmaintenance.BatchSizeDefault),
    }
}
```

Update client creation around line 864:

```go
jobCleaner := maintenance.NewJobCleaner(archetype, &maintenance.JobCleanerConfig{
    CancelledJobRetentionPeriod: config.CancelledJobRetentionPeriod,
    CompletedJobRetentionPeriod: config.CompletedJobRetentionPeriod,
    DiscardedJobRetentionPeriod: config.DiscardedJobRetentionPeriod,
    Interval:                    config.JobCleanerInterval,      // NEW
    BatchSize:                   config.JobCleanerBatchSize,     // NEW  
    QueuesExcluded:              client.pilot.JobCleanerQueuesExcluded(),
    Schema:                      config.Schema,
    Timeout:                     config.JobCleanerTimeout,
}, driver.GetExecutor())
```

### Phase 3: JobCleaner Implementation (internal/maintenance/job_cleaner.go)

Add `BatchSize` field to `JobCleanerConfig` around line 31:

```go
type JobCleanerConfig struct {
    // ... existing fields ...
    
    // BatchSize is the maximum number of jobs to delete per cleanup run.
    BatchSize int
}
```

Update validation in `mustValidate()` around line 64:

```go
func (c *JobCleanerConfig) mustValidate() *JobCleanerConfig {
    // ... existing validations ...
    if c.BatchSize <= 0 {
        panic("JobCleanerConfig.BatchSize must be above zero")
    }
    return c
}
```

Update constructor around line 98:

```go
func NewJobCleaner(archetype *baseservice.Archetype, config *JobCleanerConfig, exec riverdriver.Executor) *JobCleaner {
    return baseservice.Init(archetype, &JobCleaner{
        Config: (&JobCleanerConfig{
            // ... existing config ...
            BatchSize: cmp.Or(config.BatchSize, riversharedmaintenance.BatchSizeDefault),
        }).mustValidate(),

        batchSize: cmp.Or(config.BatchSize, riversharedmaintenance.BatchSizeDefault),
        exec:      exec,
    })
}
```

### Phase 4: Testing

#### Unit Tests (internal/maintenance/job_cleaner_test.go)

Add configuration tests:

```go
func TestJobCleaner_ConfigurableInterval(t *testing.T) {
    t.Parallel()
    
    customInterval := 5 * time.Minute
    cleaner := NewJobCleaner(
        riversharedtest.BaseServiceArchetype(t),
        &JobCleanerConfig{Interval: customInterval},
        nil,
    )
    
    require.Equal(t, customInterval, cleaner.Config.Interval)
}

func TestJobCleaner_ConfigurableBatchSize(t *testing.T) {
    t.Parallel()
    
    customBatchSize := 2500
    cleaner := NewJobCleaner(
        riversharedtest.BaseServiceArchetype(t),
        &JobCleanerConfig{BatchSize: customBatchSize},
        nil,
    )
    
    require.Equal(t, customBatchSize, cleaner.Config.BatchSize)
    require.Equal(t, customBatchSize, cleaner.batchSize)
}

func TestJobCleanerConfig_ValidatesBatchSize(t *testing.T) {
    t.Parallel()
    
    require.Panics(t, func() {
        NewJobCleaner(
            riversharedtest.BaseServiceArchetype(t),
            &JobCleanerConfig{BatchSize: 0},
            nil,
        )
    })
}
```

#### Integration Tests (client_test.go)

Add client-level configuration test:

```go
func TestNewClient_JobCleanerConfiguration(t *testing.T) {
    t.Parallel()
    
    config := &Config{
        JobCleanerInterval:  5 * time.Minute,
        JobCleanerBatchSize: 2500,
        Queues: map[string]QueueConfig{
            QueueDefault: {MaxWorkers: 1},
        },
        Workers: NewWorkers(),
    }
    
    client, _ := setup(t, config)
    
    jobCleaner := maintenance.GetService[*maintenance.JobCleaner](client.queueMaintainer)
    require.Equal(t, 5*time.Minute, jobCleaner.Config.Interval)
    require.Equal(t, 2500, jobCleaner.Config.BatchSize)
}
```

## 🔍 Code Quality Validation

### Follows River Patterns:
- **Config Structure**: Uses existing `Config` struct pattern
- **Defaults**: Uses `cmp.Or()` pattern for default values  
- **Validation**: Uses `mustValidate()` with panic for invalid config
- **Naming**: Consistent with `JobCleanerTimeout` naming
- **Documentation**: Follows River's documentation style

### Backwards Compatibility:
- All new fields are optional with sensible defaults
- Existing behavior preserved when fields not set
- No breaking changes to APIs

### Not Over-Engineered:
- **Simple approach**: Two configuration fields, nothing more
- **Leverages existing infrastructure**: Uses established config patterns
- **Minimal changes**: Only touches necessary files
- **Direct solution**: Directly addresses study's performance issues

## Implementation Steps

1. **Create branch**
2. **Implement**: Follow phases 1-4 above
3. **Test**: `make test && make lint`
4. **Validate**: Test with high-volume job database
5. **Create PR**: Include performance benchmarks from testing

## Expected Impact

- **Current**: 10.5s cleanup every 30s
- **With config**: 5-10s cleanup every 5min (or more) + higher batch sizes = much less database load
- **significant reduction in cleanup-related database load**

**General Benefits:**
- Configurable intervals reduce database contention
- Larger batch sizes improve cleanup efficiency
- Zero breaking changes ensure smooth adoption
- Follows established River patterns for consistency
