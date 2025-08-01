# Chapter 13: Scaling Strategies

## Scaling Healthcare Systems: From Local Clinic to Global Health Platform

Healthcare scaling presents unique challenges that differ significantly from typical web applications. When patient lives depend on system availability and response times, scaling strategies must account for emergency loads, regulatory compliance across jurisdictions, data sovereignty requirements, and the critical nature of healthcare services. This chapter details how we scaled MyDR24 from handling a few hundred patients to supporting millions while maintaining sub-second response times for emergency scenarios.

## Understanding Healthcare Scaling Requirements

### Healthcare-Specific Scaling Challenges

Unlike traditional applications, healthcare systems face unique scaling requirements:

1. **Life-Critical Response Times**: Emergency requests must complete within seconds regardless of system load
2. **Compliance Across Jurisdictions**: Different regions have varying healthcare regulations
3. **Data Sovereignty**: Patient data must remain within specific geographic boundaries
4. **Emergency Load Spikes**: Natural disasters or pandemics can cause 100x normal traffic
5. **Always-On Availability**: Healthcare systems cannot have maintenance windows
6. **Gradual Degradation**: Systems must degrade gracefully, prioritizing critical functions
7. **Audit Trail Scaling**: Every action must be logged, even at massive scale

### Scaling Metrics for Healthcare

We defined healthcare-specific metrics to guide our scaling decisions:

```rust
// Healthcare scaling metrics and thresholds
use std::collections::HashMap;
use serde::{Deserialize, Serialize};
use chrono::{DateTime, Utc};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct HealthcareScalingMetrics {
    // Performance metrics
    pub emergency_response_time_p99: f64,    // Must be < 2 seconds
    pub patient_query_response_time_p95: f64, // Must be < 5 seconds
    pub system_availability: f64,             // Must be > 99.9%
    
    // Load metrics
    pub concurrent_users: u64,
    pub api_requests_per_second: f64,
    pub database_connections_active: u32,
    pub emergency_requests_per_minute: u32,
    
    // Healthcare-specific metrics
    pub patient_records_accessed_per_hour: u64,
    pub fhir_transactions_per_second: f64,
    pub phi_encryption_operations_per_second: f64,
    pub audit_events_per_second: f64,
    
    // Resource utilization
    pub cpu_utilization: f64,
    pub memory_utilization: f64,
    pub database_cpu_utilization: f64,
    pub storage_iops: f64,
    
    // Compliance metrics
    pub audit_log_processing_lag: f64,       // Must be < 30 seconds
    pub backup_completion_time: f64,         // Must complete within SLA
    pub data_replication_lag: f64,          // Must be < 60 seconds
}

#[derive(Debug, Clone)]
pub struct HealthcareScalingThresholds {
    pub scale_out_thresholds: ScalingThresholds,
    pub scale_in_thresholds: ScalingThresholds,
    pub emergency_thresholds: EmergencyScalingThresholds,
}

#[derive(Debug, Clone)]
pub struct ScalingThresholds {
    pub cpu_threshold: f64,
    pub memory_threshold: f64,
    pub response_time_threshold: f64,
    pub error_rate_threshold: f64,
    pub queue_depth_threshold: u32,
}

#[derive(Debug, Clone)]
pub struct EmergencyScalingThresholds {
    pub emergency_request_spike: u32,        // Trigger emergency scaling
    pub system_overload_threshold: f64,      // Activate emergency protocols
    pub degraded_service_threshold: f64,     // Start graceful degradation
}

impl Default for HealthcareScalingThresholds {
    fn default() -> Self {
        Self {
            scale_out_thresholds: ScalingThresholds {
                cpu_threshold: 70.0,
                memory_threshold: 80.0,
                response_time_threshold: 3.0, // 3 seconds for healthcare
                error_rate_threshold: 1.0,    // 1% error rate
                queue_depth_threshold: 100,
            },
            scale_in_thresholds: ScalingThresholds {
                cpu_threshold: 30.0,
                memory_threshold: 40.0,
                response_time_threshold: 1.0,
                error_rate_threshold: 0.1,
                queue_depth_threshold: 10,
            },
            emergency_thresholds: EmergencyScalingThresholds {
                emergency_request_spike: 50,  // 50 emergency requests per minute
                system_overload_threshold: 95.0,
                degraded_service_threshold: 85.0,
            },
        }
    }
}

#[derive(Debug, Clone)]
pub struct HealthcareAutoScaler {
    metrics_collector: MetricsCollector,
    scaling_engine: ScalingEngine,
    compliance_validator: ComplianceValidator,
    emergency_handler: EmergencyScalingHandler,
    thresholds: HealthcareScalingThresholds,
}

impl HealthcareAutoScaler {
    pub async fn evaluate_scaling_decision(&self) -> Result<ScalingDecision, ScalingError> {
        // Collect current metrics
        let current_metrics = self.metrics_collector.collect_health_metrics().await?;
        
        // Check for emergency conditions first
        if let Some(emergency_decision) = self.check_emergency_scaling(&current_metrics).await? {
            return Ok(emergency_decision);
        }
        
        // Evaluate normal scaling conditions
        let scaling_decision = self.evaluate_normal_scaling(&current_metrics).await?;
        
        // Validate compliance requirements
        self.compliance_validator.validate_scaling_decision(&scaling_decision).await?;
        
        Ok(scaling_decision)
    }

    async fn check_emergency_scaling(
        &self,
        metrics: &HealthcareScalingMetrics,
    ) -> Result<Option<ScalingDecision>, ScalingError> {
        // Check for emergency request spikes
        if metrics.emergency_requests_per_minute > self.thresholds.emergency_thresholds.emergency_request_spike {
            tracing::warn!(
                emergency_requests = metrics.emergency_requests_per_minute,
                threshold = self.thresholds.emergency_thresholds.emergency_request_spike,
                "Emergency request spike detected, triggering emergency scaling"
            );
            
            return Ok(Some(ScalingDecision {
                decision_type: ScalingDecisionType::EmergencyScaleOut,
                target_instances: self.calculate_emergency_instances(metrics).await?,
                priority: ScalingPriority::Emergency,
                reason: "Emergency request spike detected".to_string(),
                compliance_approved: true,
                estimated_completion_time: 60, // 1 minute for emergency scaling
            }));
        }
        
        // Check for system overload
        if metrics.cpu_utilization > self.thresholds.emergency_thresholds.system_overload_threshold ||
           metrics.memory_utilization > self.thresholds.emergency_thresholds.system_overload_threshold {
            tracing::warn!(
                cpu = metrics.cpu_utilization,
                memory = metrics.memory_utilization,
                "System overload detected, triggering emergency scaling"
            );
            
            return Ok(Some(ScalingDecision {
                decision_type: ScalingDecisionType::EmergencyScaleOut,
                target_instances: self.calculate_overload_instances(metrics).await?,
                priority: ScalingPriority::Emergency,
                reason: "System overload detected".to_string(),
                compliance_approved: true,
                estimated_completion_time: 120, // 2 minutes for overload recovery
            }));
        }
        
        Ok(None)
    }

    async fn calculate_emergency_instances(
        &self,
        metrics: &HealthcareScalingMetrics,
    ) -> Result<u32, ScalingError> {
        // Calculate instances needed for emergency load
        let base_capacity = 10; // Minimum emergency capacity
        let spike_multiplier = (metrics.emergency_requests_per_minute as f64 / 10.0).ceil() as u32;
        let total_needed = base_capacity + spike_multiplier;
        
        // Cap at maximum allowed instances
        let max_instances = 100; // Configure based on infrastructure limits
        Ok(total_needed.min(max_instances))
    }
}

#[derive(Debug, Clone)]
pub enum ScalingDecisionType {
    NoAction,
    ScaleOut,
    ScaleIn,
    EmergencyScaleOut,
    GracefulDegradation,
}

#[derive(Debug, Clone)]
pub enum ScalingPriority {
    Low,
    Normal,
    High,
    Emergency,
}

#[derive(Debug, Clone)]
pub struct ScalingDecision {
    pub decision_type: ScalingDecisionType,
    pub target_instances: u32,
    pub priority: ScalingPriority,
    pub reason: String,
    pub compliance_approved: bool,
    pub estimated_completion_time: u32, // seconds
}
```

## Horizontal Scaling Architecture

### Multi-Tier Scaling Strategy

Our healthcare scaling architecture uses multiple tiers to handle different types of load:

```rust
// Multi-tier healthcare scaling architecture
#[derive(Debug, Clone)]
pub struct HealthcareScalingArchitecture {
    pub api_gateway_tier: APIGatewayTier,
    pub application_tier: ApplicationTier,
    pub data_tier: DataTier,
    pub cache_tier: CacheTier,
    pub emergency_tier: EmergencyTier,
}

#[derive(Debug, Clone)]
pub struct APIGatewayTier {
    pub load_balancers: Vec<LoadBalancer>,
    pub rate_limiters: Vec<RateLimiter>,
    pub health_checkers: Vec<HealthChecker>,
    pub routing_rules: RoutingConfiguration,
}

#[derive(Debug, Clone)]
pub struct ApplicationTier {
    pub general_workloads: WorkloadConfiguration,
    pub emergency_workloads: WorkloadConfiguration,
    pub background_workloads: WorkloadConfiguration,
    pub compliance_workloads: WorkloadConfiguration,
}

#[derive(Debug, Clone)]
pub struct DataTier {
    pub primary_database: DatabaseConfiguration,
    pub read_replicas: Vec<DatabaseConfiguration>,
    pub sharding_strategy: ShardingConfiguration,
    pub backup_systems: Vec<BackupConfiguration>,
}

impl HealthcareScalingArchitecture {
    pub fn production_config() -> Self {
        Self {
            api_gateway_tier: APIGatewayTier {
                load_balancers: vec![
                    LoadBalancer {
                        name: "primary-alb".to_string(),
                        algorithm: LoadBalancingAlgorithm::WeightedRoundRobin,
                        health_check_enabled: true,
                        sticky_sessions: false, // Healthcare prefers stateless
                        ssl_termination: true,
                        waf_enabled: true,
                    },
                    LoadBalancer {
                        name: "emergency-alb".to_string(),
                        algorithm: LoadBalancingAlgorithm::LeastConnections,
                        health_check_enabled: true,
                        sticky_sessions: false,
                        ssl_termination: true,
                        waf_enabled: true,
                    },
                ],
                rate_limiters: vec![
                    RateLimiter {
                        name: "general-api".to_string(),
                        requests_per_minute: 10000,
                        burst_allowance: 1000,
                        emergency_bypass: true,
                    },
                    RateLimiter {
                        name: "emergency-api".to_string(),
                        requests_per_minute: 50000,
                        burst_allowance: 5000,
                        emergency_bypass: false, // Never limit emergency requests
                    },
                ],
                health_checkers: vec![
                    HealthChecker {
                        endpoint: "/health/live".to_string(),
                        interval_seconds: 10,
                        timeout_seconds: 5,
                        healthy_threshold: 2,
                        unhealthy_threshold: 3,
                    },
                ],
                routing_rules: RoutingConfiguration {
                    emergency_priority: true,
                    compliance_routing: true,
                    geographic_routing: true,
                    version_routing: true,
                },
            },
            application_tier: ApplicationTier {
                general_workloads: WorkloadConfiguration {
                    min_instances: 6,
                    max_instances: 100,
                    target_cpu_utilization: 70.0,
                    target_memory_utilization: 80.0,
                    scaling_cooldown: 300, // 5 minutes
                    instance_type: "m6i.2xlarge".to_string(),
                },
                emergency_workloads: WorkloadConfiguration {
                    min_instances: 3,
                    max_instances: 50,
                    target_cpu_utilization: 60.0, // Lower threshold for emergency
                    target_memory_utilization: 70.0,
                    scaling_cooldown: 60, // 1 minute for emergency
                    instance_type: "c6i.4xlarge".to_string(), // High-performance instances
                },
                background_workloads: WorkloadConfiguration {
                    min_instances: 2,
                    max_instances: 20,
                    target_cpu_utilization: 80.0,
                    target_memory_utilization: 85.0,
                    scaling_cooldown: 600, // 10 minutes
                    instance_type: "m6i.large".to_string(),
                },
                compliance_workloads: WorkloadConfiguration {
                    min_instances: 2,
                    max_instances: 10,
                    target_cpu_utilization: 70.0,
                    target_memory_utilization: 80.0,
                    scaling_cooldown: 300,
                    instance_type: "m6i.xlarge".to_string(),
                },
            },
            data_tier: DataTier {
                primary_database: DatabaseConfiguration {
                    instance_type: "db.r6g.4xlarge".to_string(),
                    multi_az: true,
                    backup_retention: 35, // 5 weeks
                    monitoring_enabled: true,
                    performance_insights: true,
                    auto_scaling_enabled: true,
                },
                read_replicas: vec![
                    DatabaseConfiguration {
                        instance_type: "db.r6g.2xlarge".to_string(),
                        multi_az: false,
                        backup_retention: 7,
                        monitoring_enabled: true,
                        performance_insights: true,
                        auto_scaling_enabled: true,
                    },
                    DatabaseConfiguration {
                        instance_type: "db.r6g.2xlarge".to_string(),
                        multi_az: false,
                        backup_retention: 7,
                        monitoring_enabled: true,
                        performance_insights: true,
                        auto_scaling_enabled: true,
                    },
                ],
                sharding_strategy: ShardingConfiguration {
                    sharding_enabled: true,
                    shard_key: ShardKey::PatientRegion,
                    shard_count: 8,
                    rebalancing_enabled: true,
                },
                backup_systems: vec![
                    BackupConfiguration {
                        backup_type: BackupType::Continuous,
                        retention_days: 2555, // 7 years
                        encryption_enabled: true,
                        cross_region_replication: true,
                    },
                ],
            },
            cache_tier: CacheTier {
                redis_clusters: vec![
                    RedisClusterConfiguration {
                        cluster_name: "patient-cache".to_string(),
                        node_type: "cache.r6g.2xlarge".to_string(),
                        num_cache_nodes: 6,
                        auto_scaling_enabled: true,
                        backup_enabled: true,
                        encryption_in_transit: true,
                        encryption_at_rest: true,
                    },
                    RedisClusterConfiguration {
                        cluster_name: "session-cache".to_string(),
                        node_type: "cache.r6g.xlarge".to_string(),
                        num_cache_nodes: 3,
                        auto_scaling_enabled: true,
                        backup_enabled: true,
                        encryption_in_transit: true,
                        encryption_at_rest: true,
                    },
                ],
                cdn_configuration: CDNConfiguration {
                    enabled: true,
                    cache_behaviors: vec![
                        CacheBehavior {
                            path_pattern: "/static/*".to_string(),
                            ttl: 86400, // 24 hours
                            compress: true,
                        },
                        CacheBehavior {
                            path_pattern: "/api/public/*".to_string(),
                            ttl: 300, // 5 minutes
                            compress: true,
                        },
                    ],
                    geographic_restrictions: GeographicRestrictions {
                        restriction_type: RestrictionType::Whitelist,
                        countries: vec!["US".to_string(), "CA".to_string()], // Healthcare data sovereignty
                    },
                },
            },
            emergency_tier: EmergencyTier {
                dedicated_instances: 5,
                reserved_capacity: ReservedCapacity {
                    cpu_cores: 100,
                    memory_gb: 500,
                    storage_gb: 1000,
                },
                failover_configuration: FailoverConfiguration {
                    automatic_failover: true,
                    failover_threshold: 2.0, // 2 seconds response time
                    recovery_time_objective: 60, // 1 minute RTO
                    recovery_point_objective: 30, // 30 seconds RPO
                },
            },
        }
    }
}

#[derive(Debug, Clone)]
pub struct EmergencyTier {
    pub dedicated_instances: u32,
    pub reserved_capacity: ReservedCapacity,
    pub failover_configuration: FailoverConfiguration,
}

#[derive(Debug, Clone)]
pub struct ReservedCapacity {
    pub cpu_cores: u32,
    pub memory_gb: u32,
    pub storage_gb: u32,
}
```

## Database Scaling for Healthcare

### Sharding Strategy for Patient Data

Healthcare data requires careful sharding to maintain compliance and performance:

```rust
// Healthcare-specific database sharding
use std::collections::HashMap;
use serde::{Deserialize, Serialize};
use uuid::Uuid;

#[derive(Debug, Clone)]
pub struct HealthcareDatabaseSharding {
    pub sharding_strategy: ShardingStrategy,
    pub shard_manager: ShardManager,
    pub rebalancer: ShardRebalancer,
    pub compliance_manager: ComplianceManager,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ShardingStrategy {
    PatientRegion,      // Shard by patient's region for data sovereignty
    ProviderNetwork,    // Shard by healthcare provider network
    DataType,          // Shard by type of healthcare data
    Hybrid,            // Combination of multiple strategies
}

#[derive(Debug, Clone)]
pub struct ShardManager {
    pub shards: HashMap<String, ShardConfiguration>,
    pub routing_table: RoutingTable,
    pub metadata_store: MetadataStore,
}

#[derive(Debug, Clone)]
pub struct ShardConfiguration {
    pub shard_id: String,
    pub database_url: String,
    pub region: String,
    pub shard_key_range: ShardKeyRange,
    pub read_replicas: Vec<String>,
    pub backup_configuration: BackupConfiguration,
    pub compliance_requirements: ComplianceRequirements,
}

impl HealthcareDatabaseSharding {
    pub fn new() -> Self {
        let shards = HashMap::from([
            ("us-east-patients".to_string(), ShardConfiguration {
                shard_id: "us-east-patients".to_string(),
                database_url: "postgresql://primary-us-east.amazonaws.com:5432/patients".to_string(),
                region: "us-east-1".to_string(),
                shard_key_range: ShardKeyRange::Region("us-east".to_string()),
                read_replicas: vec![
                    "postgresql://replica1-us-east.amazonaws.com:5432/patients".to_string(),
                    "postgresql://replica2-us-east.amazonaws.com:5432/patients".to_string(),
                ],
                backup_configuration: BackupConfiguration {
                    backup_type: BackupType::Continuous,
                    retention_days: 2555,
                    encryption_enabled: true,
                    cross_region_replication: true,
                },
                compliance_requirements: ComplianceRequirements {
                    hipaa_requirements: HIPAARequirements {
                        backup_encryption: true,
                        audit_trail_preservation: true,
                        breach_notification_procedures: true,
                        business_associate_compliance: true,
                    },
                    data_retention_requirements: DataRetentionRequirements {
                        patient_records: 2555,
                        audit_logs: 2555,
                        backup_retention: 2555,
                    },
                    regulatory_reporting: RegulatoryReporting {
                        incident_reporting_required: true,
                        recovery_documentation_required: true,
                        compliance_validation_required: true,
                    },
                },
            }),
            ("us-west-patients".to_string(), ShardConfiguration {
                shard_id: "us-west-patients".to_string(),
                database_url: "postgresql://primary-us-west.amazonaws.com:5432/patients".to_string(),
                region: "us-west-2".to_string(),
                shard_key_range: ShardKeyRange::Region("us-west".to_string()),
                read_replicas: vec![
                    "postgresql://replica1-us-west.amazonaws.com:5432/patients".to_string(),
                    "postgresql://replica2-us-west.amazonaws.com:5432/patients".to_string(),
                ],
                backup_configuration: BackupConfiguration {
                    backup_type: BackupType::Continuous,
                    retention_days: 2555,
                    encryption_enabled: true,
                    cross_region_replication: true,
                },
                compliance_requirements: ComplianceRequirements {
                    hipaa_requirements: HIPAARequirements {
                        backup_encryption: true,
                        audit_trail_preservation: true,
                        breach_notification_procedures: true,
                        business_associate_compliance: true,
                    },
                    data_retention_requirements: DataRetentionRequirements {
                        patient_records: 2555,
                        audit_logs: 2555,
                        backup_retention: 2555,
                    },
                    regulatory_reporting: RegulatoryReporting {
                        incident_reporting_required: true,
                        recovery_documentation_required: true,
                        compliance_validation_required: true,
                    },
                },
            }),
        ]);

        Self {
            sharding_strategy: ShardingStrategy::PatientRegion,
            shard_manager: ShardManager {
                shards,
                routing_table: RoutingTable::new(),
                metadata_store: MetadataStore::new(),
            },
            rebalancer: ShardRebalancer::new(),
            compliance_manager: ComplianceManager::new(),
        }
    }

    pub async fn route_patient_query(
        &self,
        patient_id: Uuid,
        query_type: QueryType,
    ) -> Result<ShardTarget, ShardingError> {
        // Determine appropriate shard based on patient location
        let patient_region = self.get_patient_region(patient_id).await?;
        
        // Check if this is an emergency query requiring special routing
        if matches!(query_type, QueryType::Emergency) {
            return self.route_emergency_query(patient_id, patient_region).await;
        }
        
        // Route to appropriate shard based on region
        let shard_id = match patient_region.as_str() {
            "us-east" => "us-east-patients",
            "us-west" => "us-west-patients",
            "eu-west" => "eu-west-patients",
            "ap-south" => "ap-south-patients",
            _ => "us-east-patients", // Default fallback
        };
        
        let shard_config = self.shard_manager.shards.get(shard_id)
            .ok_or(ShardingError::ShardNotFound(shard_id.to_string()))?;
        
        // Determine read vs write target
        let target = match query_type {
            QueryType::Read | QueryType::Analytics => {
                // Use read replica for read queries
                ShardTarget::ReadReplica {
                    shard_id: shard_id.to_string(),
                    replica_url: self.select_best_replica(shard_config).await?,
                }
            }
            QueryType::Write | QueryType::Emergency => {
                // Use primary for writes and emergency queries
                ShardTarget::Primary {
                    shard_id: shard_id.to_string(),
                    primary_url: shard_config.database_url.clone(),
                }
            }
        };
        
        // Log routing decision for audit
        self.log_routing_decision(patient_id, &target, &query_type).await?;
        
        Ok(target)
    }

    async fn route_emergency_query(
        &self,
        patient_id: Uuid,
        patient_region: String,
    ) -> Result<ShardTarget, ShardingError> {
        // Emergency queries always go to primary with highest priority
        let shard_id = format!("{}-patients", patient_region);
        
        let shard_config = self.shard_manager.shards.get(&shard_id)
            .ok_or(ShardingError::ShardNotFound(shard_id.clone()))?;
        
        // Log emergency access for compliance
        self.compliance_manager.log_emergency_access(
            patient_id,
            &shard_id,
            chrono::Utc::now(),
        ).await?;
        
        Ok(ShardTarget::Emergency {
            shard_id,
            primary_url: shard_config.database_url.clone(),
            priority: EmergencyPriority::Critical,
        })
    }

    async fn select_best_replica(
        &self,
        shard_config: &ShardConfiguration,
    ) -> Result<String, ShardingError> {
        // Select replica based on current load and health
        let mut best_replica = shard_config.read_replicas[0].clone();
        let mut best_score = f64::MAX;
        
        for replica_url in &shard_config.read_replicas {
            let health_score = self.get_replica_health_score(replica_url).await?;
            if health_score < best_score {
                best_score = health_score;
                best_replica = replica_url.clone();
            }
        }
        
        Ok(best_replica)
    }

    async fn get_replica_health_score(&self, replica_url: &str) -> Result<f64, ShardingError> {
        // Calculate health score based on multiple factors
        let connection_count = self.get_connection_count(replica_url).await?;
        let response_time = self.measure_response_time(replica_url).await?;
        let cpu_utilization = self.get_cpu_utilization(replica_url).await?;
        
        // Weighted score calculation
        let score = (connection_count as f64 * 0.3) + 
                   (response_time * 0.4) + 
                   (cpu_utilization * 0.3);
        
        Ok(score)
    }

    pub async fn rebalance_shards(&self) -> Result<RebalanceResult, ShardingError> {
        tracing::info!("Starting shard rebalancing for healthcare data");
        
        // Analyze current shard distribution
        let distribution_analysis = self.analyze_shard_distribution().await?;
        
        // Check if rebalancing is needed
        if !distribution_analysis.needs_rebalancing {
            return Ok(RebalanceResult {
                rebalanced: false,
                reason: "Shards are well balanced".to_string(),
                duration: std::time::Duration::from_secs(0),
            });
        }
        
        // Plan rebalancing to maintain compliance
        let rebalance_plan = self.create_rebalance_plan(&distribution_analysis).await?;
        
        // Validate plan doesn't violate data sovereignty
        self.compliance_manager.validate_rebalance_plan(&rebalance_plan).await?;
        
        // Execute rebalancing in phases to minimize impact
        let start_time = std::time::Instant::now();
        
        for phase in rebalance_plan.phases {
            self.execute_rebalance_phase(&phase).await?;
            
            // Validate data integrity after each phase
            self.validate_phase_completion(&phase).await?;
            
            // Brief pause between phases
            tokio::time::sleep(tokio::time::Duration::from_secs(30)).await;
        }
        
        let duration = start_time.elapsed();
        
        tracing::info!(
            duration_seconds = duration.as_secs(),
            "Shard rebalancing completed successfully"
        );
        
        Ok(RebalanceResult {
            rebalanced: true,
            reason: "Rebalanced for optimal performance".to_string(),
            duration,
        })
    }
}

#[derive(Debug, Clone)]
pub enum ShardTarget {
    Primary {
        shard_id: String,
        primary_url: String,
    },
    ReadReplica {
        shard_id: String,
        replica_url: String,
    },
    Emergency {
        shard_id: String,
        primary_url: String,
        priority: EmergencyPriority,
    },
}

#[derive(Debug, Clone)]
pub enum QueryType {
    Read,
    Write,
    Emergency,
    Analytics,
}

#[derive(Debug, Clone)]
pub enum EmergencyPriority {
    Critical,
    High,
    Medium,
}

#[derive(Debug, Clone)]
pub enum ShardKeyRange {
    Region(String),
    HashRange { start: u64, end: u64 },
    Custom(String),
}
```

## Caching Strategy for Healthcare

### Multi-Layer Caching for Patient Data

Healthcare caching must balance performance with data freshness and compliance:

```rust
// Healthcare-optimized caching strategy
use std::collections::HashMap;
use std::time::Duration;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone)]
pub struct HealthcareCacheManager {
    pub l1_cache: MemoryCache,        // Application-level cache
    pub l2_cache: RedisCache,         // Distributed cache
    pub l3_cache: DatabaseCache,      // Database query cache
    pub cdn_cache: CDNCache,          // Content delivery cache
    pub cache_policies: CachePolicies,
}

#[derive(Debug, Clone)]
pub struct CachePolicies {
    pub patient_data_policy: CachePolicy,
    pub medical_records_policy: CachePolicy,
    pub provider_data_policy: CachePolicy,
    pub emergency_data_policy: CachePolicy,
    pub static_content_policy: CachePolicy,
}

#[derive(Debug, Clone)]
pub struct CachePolicy {
    pub ttl: Duration,
    pub max_size: usize,
    pub eviction_strategy: EvictionStrategy,
    pub encryption_required: bool,
    pub audit_access: bool,
    pub compliance_level: ComplianceLevel,
}

#[derive(Debug, Clone)]
pub enum EvictionStrategy {
    LRU,     // Least Recently Used
    LFU,     // Least Frequently Used
    FIFO,    // First In, First Out
    TTL,     // Time To Live based
    Healthcare, // Healthcare-specific (never evict emergency data)
}

#[derive(Debug, Clone)]
pub enum ComplianceLevel {
    Public,     // No PHI, can be cached freely
    Internal,   // Internal data, standard caching
    PHI,        // Contains PHI, strict caching rules
    Emergency,  // Emergency data, special handling
}

impl Default for CachePolicies {
    fn default() -> Self {
        Self {
            patient_data_policy: CachePolicy {
                ttl: Duration::from_secs(300), // 5 minutes for patient data
                max_size: 10000,
                eviction_strategy: EvictionStrategy::Healthcare,
                encryption_required: true,
                audit_access: true,
                compliance_level: ComplianceLevel::PHI,
            },
            medical_records_policy: CachePolicy {
                ttl: Duration::from_secs(180), // 3 minutes for medical records
                max_size: 5000,
                eviction_strategy: EvictionStrategy::LRU,
                encryption_required: true,
                audit_access: true,
                compliance_level: ComplianceLevel::PHI,
            },
            provider_data_policy: CachePolicy {
                ttl: Duration::from_secs(1800), // 30 minutes for provider data
                max_size: 1000,
                eviction_strategy: EvictionStrategy::LFU,
                encryption_required: true,
                audit_access: true,
                compliance_level: ComplianceLevel::Internal,
            },
            emergency_data_policy: CachePolicy {
                ttl: Duration::from_secs(60), // 1 minute for emergency data
                max_size: 1000,
                eviction_strategy: EvictionStrategy::Healthcare, // Never evict
                encryption_required: true,
                audit_access: true,
                compliance_level: ComplianceLevel::Emergency,
            },
            static_content_policy: CachePolicy {
                ttl: Duration::from_secs(86400), // 24 hours for static content
                max_size: 100000,
                eviction_strategy: EvictionStrategy::LRU,
                encryption_required: false,
                audit_access: false,
                compliance_level: ComplianceLevel::Public,
            },
        }
    }
}

impl HealthcareCacheManager {
    pub async fn get_patient_data(
        &self,
        patient_id: uuid::Uuid,
        request_context: RequestContext,
    ) -> Result<Option<PatientData>, CacheError> {
        let cache_key = format!("patient:{}", patient_id);
        
        // Check L1 cache first (fastest)
        if let Some(data) = self.l1_cache.get(&cache_key).await? {
            self.audit_cache_access(&cache_key, CacheLevel::L1, &request_context).await?;
            return Ok(Some(data));
        }
        
        // Check L2 cache (Redis)
        if let Some(data) = self.l2_cache.get(&cache_key).await? {
            // Populate L1 cache for faster future access
            self.l1_cache.set(&cache_key, &data, self.cache_policies.patient_data_policy.ttl).await?;
            self.audit_cache_access(&cache_key, CacheLevel::L2, &request_context).await?;
            return Ok(Some(data));
        }
        
        // Check L3 cache (database cache)
        if let Some(data) = self.l3_cache.get(&cache_key).await? {
            // Populate both L1 and L2 caches
            self.l2_cache.set(&cache_key, &data, self.cache_policies.patient_data_policy.ttl).await?;
            self.l1_cache.set(&cache_key, &data, self.cache_policies.patient_data_policy.ttl).await?;
            self.audit_cache_access(&cache_key, CacheLevel::L3, &request_context).await?;
            return Ok(Some(data));
        }
        
        // Data not in any cache
        Ok(None)
    }

    pub async fn set_patient_data(
        &self,
        patient_id: uuid::Uuid,
        data: &PatientData,
        request_context: RequestContext,
    ) -> Result<(), CacheError> {
        let cache_key = format!("patient:{}", patient_id);
        let policy = &self.cache_policies.patient_data_policy;
        
        // Encrypt data if required
        let cached_data = if policy.encryption_required {
            self.encrypt_cache_data(data).await?
        } else {
            data.clone()
        };
        
        // Set in all cache levels
        self.l1_cache.set(&cache_key, &cached_data, policy.ttl).await?;
        self.l2_cache.set(&cache_key, &cached_data, policy.ttl).await?;
        self.l3_cache.set(&cache_key, &cached_data, policy.ttl).await?;
        
        // Audit cache write for PHI data
        if policy.audit_access {
            self.audit_cache_write(&cache_key, &request_context).await?;
        }
        
        Ok(())
    }

    pub async fn invalidate_patient_cache(
        &self,
        patient_id: uuid::Uuid,
        reason: InvalidationReason,
    ) -> Result<(), CacheError> {
        let cache_key = format!("patient:{}", patient_id);
        
        // Invalidate from all cache levels
        self.l1_cache.invalidate(&cache_key).await?;
        self.l2_cache.invalidate(&cache_key).await?;
        self.l3_cache.invalidate(&cache_key).await?;
        
        // Log invalidation for audit
        tracing::info!(
            patient_id = %patient_id,
            reason = ?reason,
            cache_key = %cache_key,
            "Patient cache invalidated"
        );
        
        // If this is due to emergency update, also invalidate related caches
        if matches!(reason, InvalidationReason::EmergencyUpdate) {
            self.invalidate_related_emergency_caches(patient_id).await?;
        }
        
        Ok(())
    }

    async fn invalidate_related_emergency_caches(
        &self,
        patient_id: uuid::Uuid,
    ) -> Result<(), CacheError> {
        // Invalidate emergency-related caches
        let related_keys = vec![
            format!("emergency:patient:{}", patient_id),
            format!("medical_history:{}", patient_id),
            format!("active_treatments:{}", patient_id),
            format!("emergency_contacts:{}", patient_id),
        ];
        
        for key in related_keys {
            self.l1_cache.invalidate(&key).await?;
            self.l2_cache.invalidate(&key).await?;
            self.l3_cache.invalidate(&key).await?;
        }
        
        Ok(())
    }

    pub async fn warm_emergency_cache(&self) -> Result<(), CacheError> {
        tracing::info!("Warming emergency cache with critical patient data");
        
        // Get list of patients with active emergency flags
        let emergency_patients = self.get_emergency_patients().await?;
        
        for patient_id in emergency_patients {
            // Pre-load critical emergency data
            let emergency_data = self.load_emergency_data(patient_id).await?;
            
            let cache_key = format!("emergency:patient:{}", patient_id);
            
            // Cache with emergency policy (longer TTL, higher priority)
            self.l1_cache.set(
                &cache_key,
                &emergency_data,
                self.cache_policies.emergency_data_policy.ttl,
            ).await?;
            
            self.l2_cache.set(
                &cache_key,
                &emergency_data,
                self.cache_policies.emergency_data_policy.ttl,
            ).await?;
        }
        
        tracing::info!(
            count = emergency_patients.len(),
            "Emergency cache warming completed"
        );
        
        Ok(())
    }

    async fn audit_cache_access(
        &self,
        cache_key: &str,
        cache_level: CacheLevel,
        request_context: &RequestContext,
    ) -> Result<(), CacheError> {
        // Audit PHI cache access for compliance
        if cache_key.contains("patient:") || cache_key.contains("medical:") {
            tracing::debug!(
                cache_key = %cache_key,
                cache_level = ?cache_level,
                user_id = %request_context.user_id,
                "PHI cache access"
            );
            
            // Log to audit system
            // Implementation would integrate with audit logging system
        }
        
        Ok(())
    }
}

#[derive(Debug, Clone)]
pub enum CacheLevel {
    L1,  // Memory cache
    L2,  // Redis cache
    L3,  // Database cache
    CDN, // Content delivery network
}

#[derive(Debug, Clone)]
pub enum InvalidationReason {
    DataUpdate,
    EmergencyUpdate,
    ComplianceRequirement,
    UserRequest,
    SystemMaintenance,
}

#[derive(Debug, Clone)]
pub struct RequestContext {
    pub user_id: uuid::Uuid,
    pub session_id: String,
    pub request_id: String,
    pub ip_address: String,
    pub user_agent: String,
    pub access_time: chrono::DateTime<chrono::Utc>,
}
```

## Geographic Scaling

### Multi-Region Healthcare Deployment

Healthcare systems must consider data sovereignty and regulatory requirements:

```rust
// Geographic scaling for healthcare compliance
#[derive(Debug, Clone)]
pub struct HealthcareGeographicScaling {
    pub regions: HashMap<String, RegionConfiguration>,
    pub data_sovereignty_rules: DataSovereigntyRules,
    pub cross_region_replication: CrossRegionReplication,
    pub compliance_manager: ComplianceManager,
}

#[derive(Debug, Clone)]
pub struct RegionConfiguration {
    pub region_id: String,
    pub region_name: String,
    pub regulatory_framework: RegulatoryFramework,
    pub data_residency_required: bool,
    pub healthcare_authorities: Vec<HealthcareAuthority>,
    pub infrastructure: RegionalInfrastructure,
}

#[derive(Debug, Clone)]
pub enum RegulatoryFramework {
    HIPAA,      // United States
    GDPR,       // European Union
    PIPEDA,     // Canada
    PDPA,       // Singapore
    Combined(Vec<RegulatoryFramework>), // Multiple frameworks
}

impl HealthcareGeographicScaling {
    pub fn global_deployment() -> Self {
        let mut regions = HashMap::new();
        
        // United States deployment
        regions.insert("us-east-1".to_string(), RegionConfiguration {
            region_id: "us-east-1".to_string(),
            region_name: "US East (N. Virginia)".to_string(),
            regulatory_framework: RegulatoryFramework::HIPAA,
            data_residency_required: true,
            healthcare_authorities: vec![
                HealthcareAuthority {
                    name: "HHS".to_string(),
                    contact: "hhs@gov.us".to_string(),
                    reporting_requirements: vec![
                        "HIPAA breach notification".to_string(),
                        "Quality reporting".to_string(),
                    ],
                },
                HealthcareAuthority {
                    name: "FDA".to_string(),
                    contact: "fda@gov.us".to_string(),
                    reporting_requirements: vec![
                        "Medical device reporting".to_string(),
                        "Adverse event reporting".to_string(),
                    ],
                },
            ],
            infrastructure: RegionalInfrastructure {
                availability_zones: vec!["us-east-1a".to_string(), "us-east-1b".to_string(), "us-east-1c".to_string()],
                instance_types: vec!["m6i.2xlarge".to_string(), "c6i.4xlarge".to_string()],
                database_config: DatabaseConfig {
                    primary_az: "us-east-1a".to_string(),
                    replica_azs: vec!["us-east-1b".to_string(), "us-east-1c".to_string()],
                    backup_region: Some("us-west-2".to_string()),
                },
                networking: NetworkingConfig {
                    vpc_cidr: "10.0.0.0/16".to_string(),
                    private_subnets: vec!["10.0.1.0/24".to_string(), "10.0.2.0/24".to_string()],
                    public_subnets: vec!["10.0.101.0/24".to_string(), "10.0.102.0/24".to_string()],
                },
            },
        });
        
        // European Union deployment
        regions.insert("eu-west-1".to_string(), RegionConfiguration {
            region_id: "eu-west-1".to_string(),
            region_name: "EU West (Ireland)".to_string(),
            regulatory_framework: RegulatoryFramework::GDPR,
            data_residency_required: true,
            healthcare_authorities: vec![
                HealthcareAuthority {
                    name: "EMA".to_string(),
                    contact: "ema@europa.eu".to_string(),
                    reporting_requirements: vec![
                        "GDPR compliance reporting".to_string(),
                        "Medical device regulation".to_string(),
                    ],
                },
            ],
            infrastructure: RegionalInfrastructure {
                availability_zones: vec!["eu-west-1a".to_string(), "eu-west-1b".to_string(), "eu-west-1c".to_string()],
                instance_types: vec!["m6i.2xlarge".to_string(), "c6i.4xlarge".to_string()],
                database_config: DatabaseConfig {
                    primary_az: "eu-west-1a".to_string(),
                    replica_azs: vec!["eu-west-1b".to_string(), "eu-west-1c".to_string()],
                    backup_region: Some("eu-central-1".to_string()),
                },
                networking: NetworkingConfig {
                    vpc_cidr: "10.1.0.0/16".to_string(),
                    private_subnets: vec!["10.1.1.0/24".to_string(), "10.1.2.0/24".to_string()],
                    public_subnets: vec!["10.1.101.0/24".to_string(), "10.1.102.0/24".to_string()],
                },
            },
        });
        
        Self {
            regions,
            data_sovereignty_rules: DataSovereigntyRules::strict(),
            cross_region_replication: CrossRegionReplication::compliant(),
            compliance_manager: ComplianceManager::new(),
        }
    }

    pub async fn route_patient_request(
        &self,
        patient_id: uuid::Uuid,
        request_origin: RequestOrigin,
    ) -> Result<RegionRoutingDecision, GeographicScalingError> {
        // Determine patient's home region
        let patient_region = self.determine_patient_region(patient_id).await?;
        
        // Check data sovereignty requirements
        let sovereignty_check = self.data_sovereignty_rules
            .validate_cross_border_access(&patient_region, &request_origin.region)
            .await?;
        
        if !sovereignty_check.allowed {
            return Err(GeographicScalingError::DataSovereigntyViolation {
                patient_region: patient_region.clone(),
                request_region: request_origin.region.clone(),
                violation_reason: sovereignty_check.reason,
            });
        }
        
        // Determine optimal region for processing
        let target_region = if sovereignty_check.must_process_locally {
            patient_region.clone()
        } else {
            self.select_optimal_processing_region(&patient_region, &request_origin).await?
        };
        
        Ok(RegionRoutingDecision {
            target_region,
            patient_home_region: patient_region,
            cross_region_allowed: sovereignty_check.allowed,
            compliance_requirements: sovereignty_check.compliance_requirements,
            routing_reason: self.explain_routing_decision(&target_region, &request_origin).await,
        })
    }

    async fn select_optimal_processing_region(
        &self,
        patient_region: &str,
        request_origin: &RequestOrigin,
    ) -> Result<String, GeographicScalingError> {
        // Collect region performance metrics
        let mut region_scores = HashMap::new();
        
        for (region_id, region_config) in &self.regions {
            let mut score = 0.0;
            
            // Latency score (lower is better)
            let latency = self.measure_latency(&request_origin.region, region_id).await?;
            score += 100.0 - latency.min(100.0);
            
            // Load score (lower is better)
            let load = self.get_region_load(region_id).await?;
            score += 100.0 - load;
            
            // Compliance score
            if region_id == patient_region {
                score += 50.0; // Bonus for patient's home region
            }
            
            // Regulatory compatibility
            if self.are_regulations_compatible(patient_region, region_id).await? {
                score += 25.0;
            }
            
            region_scores.insert(region_id.clone(), score);
        }
        
        // Select region with highest score
        let best_region = region_scores.iter()
            .max_by(|a, b| a.1.partial_cmp(b.1).unwrap())
            .map(|(region, _)| region.clone())
            .ok_or(GeographicScalingError::NoSuitableRegion)?;
        
        Ok(best_region)
    }
}

#[derive(Debug, Clone)]
pub struct DataSovereigntyRules {
    pub rules: HashMap<String, SovereigntyRule>,
    pub default_policy: SovereigntyPolicy,
}

#[derive(Debug, Clone)]
pub struct SovereigntyRule {
    pub region: String,
    pub data_must_stay_local: bool,
    pub allowed_backup_regions: Vec<String>,
    pub cross_border_restrictions: Vec<CrossBorderRestriction>,
}

impl DataSovereigntyRules {
    pub fn strict() -> Self {
        let mut rules = HashMap::new();
        
        // US healthcare data sovereignty
        rules.insert("us-east-1".to_string(), SovereigntyRule {
            region: "us-east-1".to_string(),
            data_must_stay_local: true,
            allowed_backup_regions: vec!["us-west-2".to_string()], // Only other US regions
            cross_border_restrictions: vec![
                CrossBorderRestriction {
                    prohibited_regions: vec!["eu-west-1".to_string(), "ap-south-1".to_string()],
                    reason: "HIPAA requires US data residency".to_string(),
                },
            ],
        });
        
        // EU healthcare data sovereignty (GDPR)
        rules.insert("eu-west-1".to_string(), SovereigntyRule {
            region: "eu-west-1".to_string(),
            data_must_stay_local: true,
            allowed_backup_regions: vec!["eu-central-1".to_string()], // Only other EU regions
            cross_border_restrictions: vec![
                CrossBorderRestriction {
                    prohibited_regions: vec!["us-east-1".to_string(), "ap-south-1".to_string()],
                    reason: "GDPR requires EU data residency".to_string(),
                },
            ],
        });
        
        Self {
            rules,
            default_policy: SovereigntyPolicy::StrictLocal,
        }
    }
}
```

## Performance Optimization at Scale

### Healthcare-Specific Performance Patterns

At scale, healthcare systems require specialized optimization strategies:

```rust
// Performance optimization for healthcare at scale
#[derive(Debug, Clone)]
pub struct HealthcarePerformanceOptimizer {
    pub query_optimizer: QueryOptimizer,
    pub connection_pooler: ConnectionPoolManager,
    pub request_prioritizer: RequestPrioritizer,
    pub resource_manager: ResourceManager,
}

impl HealthcarePerformanceOptimizer {
    pub async fn optimize_for_healthcare_load(&self) -> Result<OptimizationResult, OptimizationError> {
        // Optimize database queries for healthcare patterns
        let query_optimizations = self.optimize_healthcare_queries().await?;
        
        // Optimize connection pooling for healthcare workloads
        let connection_optimizations = self.optimize_connection_pools().await?;
        
        // Optimize request prioritization for emergency scenarios
        let prioritization_optimizations = self.optimize_request_prioritization().await?;
        
        // Optimize resource allocation for healthcare demands
        let resource_optimizations = self.optimize_resource_allocation().await?;
        
        Ok(OptimizationResult {
            query_optimizations,
            connection_optimizations,
            prioritization_optimizations,
            resource_optimizations,
            total_improvement: self.calculate_total_improvement().await?,
        })
    }

    async fn optimize_healthcare_queries(&self) -> Result<QueryOptimizations, OptimizationError> {
        // Healthcare-specific query optimizations
        let optimizations = vec![
            // Patient lookup optimization
            QueryOptimization {
                query_type: "patient_lookup".to_string(),
                optimization: "Add composite index on (region, last_name, first_name)".to_string(),
                estimated_improvement: 60.0, // 60% faster
                impact_level: ImpactLevel::High,
            },
            
            // Medical history optimization
            QueryOptimization {
                query_type: "medical_history".to_string(),
                optimization: "Partition by patient_id and date range".to_string(),
                estimated_improvement: 75.0,
                impact_level: ImpactLevel::High,
            },
            
            // Emergency patient search
            QueryOptimization {
                query_type: "emergency_search".to_string(),
                optimization: "Materialized view for active emergency patients".to_string(),
                estimated_improvement: 90.0,
                impact_level: ImpactLevel::Critical,
            },
            
            // Provider schedule optimization
            QueryOptimization {
                query_type: "provider_schedule".to_string(),
                optimization: "Cache frequently accessed schedules".to_string(),
                estimated_improvement: 50.0,
                impact_level: ImpactLevel::Medium,
            },
        ];
        
        Ok(QueryOptimizations {
            optimizations,
            total_queries_optimized: optimizations.len(),
            estimated_total_improvement: 68.75, // Average improvement
        })
    }

    async fn optimize_connection_pools(&self) -> Result<ConnectionOptimizations, OptimizationError> {
        // Healthcare workload patterns require specialized connection pooling
        let optimizations = ConnectionOptimizations {
            // Emergency pool - always available
            emergency_pool: PoolConfiguration {
                min_connections: 10,
                max_connections: 50,
                idle_timeout: Duration::from_secs(300), // 5 minutes
                max_lifetime: Duration::from_secs(3600), // 1 hour
                health_check_interval: Duration::from_secs(30),
                priority: PoolPriority::Emergency,
            },
            
            // General healthcare pool
            general_pool: PoolConfiguration {
                min_connections: 20,
                max_connections: 200,
                idle_timeout: Duration::from_secs(600), // 10 minutes
                max_lifetime: Duration::from_secs(7200), // 2 hours
                health_check_interval: Duration::from_secs(60),
                priority: PoolPriority::Normal,
            },
            
            // Background/analytics pool
            background_pool: PoolConfiguration {
                min_connections: 5,
                max_connections: 50,
                idle_timeout: Duration::from_secs(1800), // 30 minutes
                max_lifetime: Duration::from_secs(14400), // 4 hours
                health_check_interval: Duration::from_secs(120),
                priority: PoolPriority::Low,
            },
            
            // Read-only pool for reports
            readonly_pool: PoolConfiguration {
                min_connections: 10,
                max_connections: 100,
                idle_timeout: Duration::from_secs(900), // 15 minutes
                max_lifetime: Duration::from_secs(3600), // 1 hour
                health_check_interval: Duration::from_secs(90),
                priority: PoolPriority::Normal,
            },
        };
        
        Ok(optimizations)
    }
}

// Healthcare-specific request prioritization
#[derive(Debug, Clone)]
pub struct RequestPrioritizer {
    pub priority_rules: Vec<PriorityRule>,
    pub load_balancer: PriorityLoadBalancer,
}

#[derive(Debug, Clone)]
pub struct PriorityRule {
    pub rule_name: String,
    pub condition: PriorityCondition,
    pub priority_level: PriorityLevel,
    pub resource_allocation: ResourceAllocation,
}

#[derive(Debug, Clone)]
pub enum PriorityCondition {
    EmergencyRequest,
    CriticalPatient,
    HighVolumeProvider,
    RegularRequest,
    BackgroundTask,
}

#[derive(Debug, Clone)]
pub enum PriorityLevel {
    Emergency = 1,    // Highest priority
    Critical = 2,
    High = 3,
    Normal = 4,
    Low = 5,          // Lowest priority
}

impl RequestPrioritizer {
    pub fn healthcare_rules() -> Vec<PriorityRule> {
        vec![
            PriorityRule {
                rule_name: "Emergency Medical Requests".to_string(),
                condition: PriorityCondition::EmergencyRequest,
                priority_level: PriorityLevel::Emergency,
                resource_allocation: ResourceAllocation {
                    cpu_percentage: 30.0,
                    memory_percentage: 30.0,
                    connection_percentage: 25.0,
                },
            },
            PriorityRule {
                rule_name: "Critical Patient Access".to_string(),
                condition: PriorityCondition::CriticalPatient,
                priority_level: PriorityLevel::Critical,
                resource_allocation: ResourceAllocation {
                    cpu_percentage: 25.0,
                    memory_percentage: 25.0,
                    connection_percentage: 20.0,
                },
            },
            PriorityRule {
                rule_name: "High Volume Providers".to_string(),
                condition: PriorityCondition::HighVolumeProvider,
                priority_level: PriorityLevel::High,
                resource_allocation: ResourceAllocation {
                    cpu_percentage: 20.0,
                    memory_percentage: 20.0,
                    connection_percentage: 25.0,
                },
            },
            PriorityRule {
                rule_name: "Regular Healthcare Requests".to_string(),
                condition: PriorityCondition::RegularRequest,
                priority_level: PriorityLevel::Normal,
                resource_allocation: ResourceAllocation {
                    cpu_percentage: 20.0,
                    memory_percentage: 20.0,
                    connection_percentage: 25.0,
                },
            },
            PriorityRule {
                rule_name: "Background Processing".to_string(),
                condition: PriorityCondition::BackgroundTask,
                priority_level: PriorityLevel::Low,
                resource_allocation: ResourceAllocation {
                    cpu_percentage: 5.0,
                    memory_percentage: 5.0,
                    connection_percentage: 5.0,
                },
            },
        ]
    }
}
```

## Lessons Learned

### Scaling Insights for Healthcare

1. **Emergency-First Design**: Always reserve capacity for emergency scenarios
2. **Compliance-Aware Scaling**: Geographic scaling must respect data sovereignty
3. **Performance Degradation**: Graceful degradation prioritizes critical healthcare functions
4. **Multi-Tier Caching**: Healthcare data requires specialized caching strategies
5. **Request Prioritization**: Emergency requests must bypass normal load balancing

### Healthcare Scaling Best Practices

1. **Reserve Emergency Capacity**: Always maintain dedicated resources for emergency scenarios
2. **Regional Compliance**: Design scaling strategies around regulatory requirements
3. **Data Sovereignty**: Implement strict controls for cross-border data movement
4. **Performance Monitoring**: Healthcare-specific metrics guide scaling decisions
5. **Graceful Degradation**: Non-critical features should degrade before critical ones

Our comprehensive scaling strategy ensured MyDR24 could handle massive growth while maintaining the performance and compliance requirements essential for healthcare systems. In the next chapter, we'll explore the operational practices that keep these scaled systems running smoothly.

---

**Next Chapter**: [Operations & Maintenance](./chapter14-operations-maintenance.md) - The operational practices, monitoring strategies, and maintenance procedures that keep MyDR24 running reliably at scale.
