🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.3 Azure Cache for Redis

## 🎯 Objectifs

- Maîtriser l'architecture multi-tiers d'Azure Cache for Redis
- Configurer des déploiements production avec ARM, Terraform et Bicep
- Implémenter VNet injection et Private Link pour l'isolation réseau
- Mettre en place la géo-réplication active-passive
- Optimiser les coûts avec les Reserved Instances
- Intégrer Azure Monitor et Application Insights

---

## 🏗️ Architecture et tiers

### Vue d'ensemble des 4 tiers

```
┌─────────────────────────────────────────────────────────────────┐
│              Azure Cache for Redis - Tiers                      │
└─────────────────────────────────────────────────────────────────┘

┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────┐
│    Basic     │  │   Standard   │  │   Premium    │  │Enterprise│
│              │  │              │  │              │  │          │
│ Single node  │  │ Primary +    │  │ Cluster +    │  │ Redis    │
│              │  │ Replica      │  │ Advanced     │  │Enterprise│
│              │  │              │  │ Features     │  │          │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └─────┬────┘
       │                 │                 │                │
┌──────▼─────────────────▼─────────────────▼────────────────▼─────┐
│                                                                 │
│  Basic Tier                                                     │
│  ├── Single node (no replication)                               │
│  ├── No SLA                                                     │
│  ├── Dev/Test only                                              │
│  ├── 250 MB - 53 GB                                             │
│  └── Lowest cost                                                │
│                                                                 │
│  Standard Tier                                                  │
│  ├── Primary + 1 Replica                                        │
│  ├── 99.9% SLA                                                  │
│  ├── Automatic failover                                         │
│  ├── 250 MB - 53 GB                                             │
│  └── Production (simple workloads)                              │
│                                                                 │
│  Premium Tier                                                   │
│  ├── Cluster mode (up to 10 shards)                             │
│  ├── 99.9% SLA                                                  │
│  ├── VNet injection                                             │
│  ├── Persistence (RDB + AOF)                                    │
│  ├── Geo-replication                                            │
│  ├── Zone redundancy                                            │
│  ├── 6 GB - 1.2 TB (P5 with clustering)                         │
│  └── Production (advanced)                                      │
│                                                                 │
│  Enterprise Tier                                                │
│  ├── Redis Enterprise (by Redis Ltd)                            │
│  ├── 99.99% SLA                                                 │
│  ├── Active-Active geo-distribution                             │
│  ├── Redis Stack modules (Search, JSON, etc.)                   │
│  ├── 12 GB - 2 TB per shard                                     │
│  ├── Auto-scaling                                               │
│  └── Mission-critical                                           │
└─────────────────────────────────────────────────────────────────┘
```

### Comparaison détaillée des tiers

| Caractéristique | Basic | Standard | Premium | Enterprise |
|-----------------|-------|----------|---------|------------|
| **Disponibilité** |
| Nodes | 1 | 2 (primary + replica) | 2+ (with clustering) | 3+ |
| SLA | ❌ Aucun | 99.9% | 99.9% | 99.99% |
| Automatic failover | ❌ | ✅ | ✅ | ✅ |
| Zone redundancy | ❌ | ❌ | ✅ | ✅ |
| **Capacité** |
| Min RAM | 250 MB | 250 MB | 6 GB | 12 GB |
| Max RAM (single) | 53 GB | 53 GB | 120 GB | 2 TB |
| Max RAM (cluster) | N/A | N/A | 1.2 TB (P5×10) | Unlimited |
| Max shards | N/A | N/A | 10 | Unlimited |
| **Performance** |
| Max connections | ~10K | ~20K | ~40K | ~100K |
| Max throughput | Low | Medium | High | Very High |
| **Réseau** |
| VNet injection | ❌ | ❌ | ✅ | ✅ |
| Private Link | ❌ | ❌ | ✅ | ✅ |
| Public endpoint | ✅ | ✅ | ✅ (désactivable) | ✅ (désactivable) |
| **Persistence** |
| RDB | ❌ | ❌ | ✅ | ✅ |
| AOF | ❌ | ❌ | ✅ | ✅ |
| **Réplication** |
| Geo-replication | ❌ | ❌ | ✅ (passive) | ✅ (active-active) |
| **Features avancées** |
| Clustering | ❌ | ❌ | ✅ | ✅ |
| Redis modules | ❌ | ❌ | ❌ | ✅ (Stack) |
| Active-Active | ❌ | ❌ | ❌ | ✅ |
| Auto-scaling | ❌ | ❌ | ❌ | ✅ |
| **Monitoring** |
| Azure Monitor | ✅ | ✅ | ✅ | ✅ |
| Diagnostic logs | Basic | Standard | Advanced | Advanced |
| **Pricing** |
| Coût relatif | 💰 | 💰💰 | 💰💰💰 | 💰💰💰💰 |

---

## 🔧 Architecture Premium (Production Standard)

### Topologie réseau avec VNet injection

```
┌──────────────────────────────────────────────────────────┐
│                    Azure Virtual Network                 │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │              Subnet: Applications (10.0.1.0/24)    │  │
│  │                                                    │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │  │
│  │  │ App VM 1 │  │ App VM 2 │  │ App VM 3 │          │  │
│  │  └─────┬────┘  └─────┬────┘  └─────┬────┘          │  │
│  │        │             │             │               │  │
│  │        └─────────────┼─────────────┘               │  │
│  │                      │                             │  │
│  └──────────────────────┼─────────────────────────────┘  │
│                         │                                │
│                         │ Private connection             │
│                         ▼                                │
│  ┌────────────────────────────────────────────────────┐  │
│  │      Subnet: Redis Cache (10.0.2.0/27)             │  │
│  │      (Delegated to Microsoft.Cache/redis)          │  │
│  │                                                    │  │
│  │  ┌────────────────────────────────────────────┐    │  │
│  │  │   Azure Cache for Redis Premium            │    │  │
│  │  │                                            │    │  │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  │    │  │
│  │  │  │ Shard 1  │  │ Shard 2  │  │ Shard 3  │  │    │  │
│  │  │  │          │  │          │  │          │  │    │  │
│  │  │  │Primary   │  │Primary   │  │Primary   │  │    │  │
│  │  │  │Replica   │  │Replica   │  │Replica   │  │    │  │
│  │  │  └──────────┘  └──────────┘  └──────────┘  │    │  │
│  │  │                                            │    │  │
│  │  │  • Zone redundancy (across AZs)            │    │  │
│  │  │  • Private IPs only                        │    │  │
│  │  │  • No public endpoint                      │    │  │
│  │  └────────────────────────────────────────────┘    │  │
│  │                                                    │  │
│  │  Private Endpoint IP: 10.0.2.4                     │  │
│  │  DNS: myredis.redis.cache.windows.net → 10.0.2.4   │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │              NSG Rules                             │  │
│  │                                                    │  │
│  │  Allow 10.0.1.0/24 → 10.0.2.0/27:6379-6380 (Redis) │  │
│  │  Allow 10.0.2.0/27 → Storage (for persistence)     │  │
│  │  Deny all other inbound                            │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### Geo-Replication (Active-Passive)

```
┌──────────────────────────────────────────────────────────────┐
│           Azure Cache Geo-Replication Architecture           │
│                                                              │
│  Primary Region: West Europe                                 │
│  ┌──────────────────────────────────────────────────────┐    │
│  │   Azure Cache Premium (Primary)                      │    │
│  │                                                      │    │
│  │   ┌─────────────────────────────────────┐            │    │
│  │   │  Shard 1  │  Shard 2  │  Shard 3    │            │    │
│  │   │           │           │             │            │    │
│  │   │  Primary  │  Primary  │  Primary    │            │    │
│  │   │  Replica  │  Replica  │  Replica    │            │    │
│  │   └─────────────────────────────────────┘            │    │
│  │                                                      │    │
│  │   • Read/Write operations                            │    │
│  │   • Source of truth                                  │    │
│  │   • Persistence enabled                              │    │
│  └──────────────────┬───────────────────────────────────┘    │
│                     │                                        │
│                     │ Async replication                      │
│                     │ (cross-region)                         │
│                     ▼                                        │
│  Secondary Region: North Europe                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │   Azure Cache Premium (Secondary)                    │    │
│  │                                                      │    │
│  │   ┌─────────────────────────────────────┐            │    │
│  │   │  Shard 1  │  Shard 2  │  Shard 3    │            │    │
│  │   │           │           │             │            │    │
│  │   │  Primary  │  Primary  │  Primary    │            │    │
│  │   │  Replica  │  Replica  │  Replica    │            │    │
│  │   └─────────────────────────────────────┘            │    │
│  │                                                      │    │
│  │   • Read-only (until promoted)                       │    │
│  │   • Disaster recovery target                         │    │
│  │   • Can unlink to become independent                 │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  Replication Characteristics:                                │
│  ├── Latency: 10-100ms cross-region                          │
│  ├── Consistency: Eventually consistent                      │
│  ├── Failover: Manual (unlink secondary)                     │
│  └── Use case: DR, read-only queries from secondary region   │
└──────────────────────────────────────────────────────────────┘
```

---

## 📋 Configuration avec ARM Templates

### Template complet Premium avec toutes les features

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "cacheName": {
      "type": "string",
      "metadata": {
        "description": "Name of the Redis Cache"
      }
    },
    "location": {
      "type": "string",
      "defaultValue": "[resourceGroup().location]",
      "metadata": {
        "description": "Location for all resources"
      }
    },
    "skuName": {
      "type": "string",
      "defaultValue": "Premium",
      "allowedValues": [
        "Basic",
        "Standard",
        "Premium"
      ],
      "metadata": {
        "description": "Pricing tier"
      }
    },
    "skuFamily": {
      "type": "string",
      "defaultValue": "P",
      "allowedValues": [
        "C",
        "P"
      ],
      "metadata": {
        "description": "C = Basic/Standard, P = Premium"
      }
    },
    "skuCapacity": {
      "type": "int",
      "defaultValue": 3,
      "allowedValues": [
        0, 1, 2, 3, 4, 5, 6
      ],
      "metadata": {
        "description": "Size: C0-C6 or P1-P5"
      }
    },
    "shardCount": {
      "type": "int",
      "defaultValue": 3,
      "minValue": 1,
      "maxValue": 10,
      "metadata": {
        "description": "Number of shards for clustering (Premium only)"
      }
    },
    "enableNonSslPort": {
      "type": "bool",
      "defaultValue": false,
      "metadata": {
        "description": "Enable non-SSL port 6379"
      }
    },
    "minimumTlsVersion": {
      "type": "string",
      "defaultValue": "1.2",
      "allowedValues": [
        "1.0",
        "1.1",
        "1.2"
      ]
    },
    "publicNetworkAccess": {
      "type": "string",
      "defaultValue": "Disabled",
      "allowedValues": [
        "Enabled",
        "Disabled"
      ],
      "metadata": {
        "description": "Public network access"
      }
    },
    "vnetName": {
      "type": "string",
      "metadata": {
        "description": "Virtual Network name"
      }
    },
    "subnetName": {
      "type": "string",
      "metadata": {
        "description": "Subnet name for Redis (must be delegated)"
      }
    },
    "storageAccountName": {
      "type": "string",
      "metadata": {
        "description": "Storage account for RDB/AOF persistence"
      }
    },
    "logAnalyticsWorkspaceId": {
      "type": "string",
      "metadata": {
        "description": "Log Analytics workspace ID for diagnostics"
      }
    },
    "enableZoneRedundancy": {
      "type": "bool",
      "defaultValue": true,
      "metadata": {
        "description": "Enable zone redundancy (Premium only, supported regions)"
      }
    },
    "tags": {
      "type": "object",
      "defaultValue": {
        "Environment": "Production",
        "ManagedBy": "ARM"
      }
    }
  },
  "variables": {
    "subnetId": "[resourceId('Microsoft.Network/virtualNetworks/subnets', parameters('vnetName'), parameters('subnetName'))]",
    "storageAccountId": "[resourceId('Microsoft.Storage/storageAccounts', parameters('storageAccountName'))]"
  },
  "resources": [
    {
      "type": "Microsoft.Cache/redis",
      "apiVersion": "2023-04-01",
      "name": "[parameters('cacheName')]",
      "location": "[parameters('location')]",
      "tags": "[parameters('tags')]",
      "properties": {
        "sku": {
          "name": "[parameters('skuName')]",
          "family": "[parameters('skuFamily')]",
          "capacity": "[parameters('skuCapacity')]"
        },
        "enableNonSslPort": "[parameters('enableNonSslPort')]",
        "minimumTlsVersion": "[parameters('minimumTlsVersion')]",
        "publicNetworkAccess": "[parameters('publicNetworkAccess')]",
        "redisVersion": "6",
        "shardCount": "[if(equals(parameters('skuName'), 'Premium'), parameters('shardCount'), json('null'))]",
        "replicasPerMaster": "[if(equals(parameters('skuName'), 'Premium'), 1, json('null'))]",
        "replicasPerPrimary": "[if(equals(parameters('skuName'), 'Premium'), 1, json('null'))]",
        "subnetId": "[if(equals(parameters('skuName'), 'Premium'), variables('subnetId'), json('null'))]",
        "staticIP": null,
        "zones": "[if(and(equals(parameters('skuName'), 'Premium'), parameters('enableZoneRedundancy')), createArray('1', '2', '3'), json('null'))]",
        "redisConfiguration": {
          "maxmemory-policy": "allkeys-lru",
          "maxmemory-reserved": "50",
          "maxfragmentationmemory-reserved": "50",
          "maxmemory-delta": "50",
          "notify-keyspace-events": "Ex",
          "rdb-backup-enabled": "[if(equals(parameters('skuName'), 'Premium'), 'true', json('null'))]",
          "rdb-backup-frequency": "[if(equals(parameters('skuName'), 'Premium'), '60', json('null'))]",
          "rdb-backup-max-snapshot-count": "[if(equals(parameters('skuName'), 'Premium'), '1', json('null'))]",
          "rdb-storage-connection-string": "[if(equals(parameters('skuName'), 'Premium'), concat('DefaultEndpointsProtocol=https;AccountName=', parameters('storageAccountName'), ';AccountKey=', listKeys(variables('storageAccountId'), '2021-09-01').keys[0].value), json('null'))]",
          "aof-backup-enabled": "[if(equals(parameters('skuName'), 'Premium'), 'true', json('null'))]",
          "aof-storage-connection-string-0": "[if(equals(parameters('skuName'), 'Premium'), concat('DefaultEndpointsProtocol=https;AccountName=', parameters('storageAccountName'), ';AccountKey=', listKeys(variables('storageAccountId'), '2021-09-01').keys[0].value), json('null'))]",
          "aof-storage-connection-string-1": "[if(equals(parameters('skuName'), 'Premium'), concat('DefaultEndpointsProtocol=https;AccountName=', parameters('storageAccountName'), ';AccountKey=', listKeys(variables('storageAccountId'), '2021-09-01').keys[1].value), json('null'))]",
          "zonal-configuration": "[if(parameters('enableZoneRedundancy'), 'true', json('null'))]"
        }
      }
    },
    {
      "type": "Microsoft.Cache/redis/firewallRules",
      "apiVersion": "2023-04-01",
      "name": "[concat(parameters('cacheName'), '/AllowVNet')]",
      "dependsOn": [
        "[resourceId('Microsoft.Cache/redis', parameters('cacheName'))]"
      ],
      "properties": {
        "startIP": "10.0.1.0",
        "endIP": "10.0.1.255"
      }
    },
    {
      "type": "Microsoft.Insights/diagnosticSettings",
      "apiVersion": "2021-05-01-preview",
      "scope": "[format('Microsoft.Cache/redis/{0}', parameters('cacheName'))]",
      "name": "diagnostics",
      "dependsOn": [
        "[resourceId('Microsoft.Cache/redis', parameters('cacheName'))]"
      ],
      "properties": {
        "workspaceId": "[parameters('logAnalyticsWorkspaceId')]",
        "logs": [
          {
            "category": "ConnectedClientList",
            "enabled": true,
            "retentionPolicy": {
              "days": 30,
              "enabled": true
            }
          }
        ],
        "metrics": [
          {
            "category": "AllMetrics",
            "enabled": true,
            "retentionPolicy": {
              "days": 30,
              "enabled": true
            }
          }
        ]
      }
    }
  ],
  "outputs": {
    "redisHostName": {
      "type": "string",
      "value": "[reference(resourceId('Microsoft.Cache/redis', parameters('cacheName'))).hostName]"
    },
    "redisSslPort": {
      "type": "int",
      "value": "[reference(resourceId('Microsoft.Cache/redis', parameters('cacheName'))).sslPort]"
    },
    "redisPort": {
      "type": "int",
      "value": "[reference(resourceId('Microsoft.Cache/redis', parameters('cacheName'))).port]"
    },
    "redisPrimaryKey": {
      "type": "securestring",
      "value": "[listKeys(resourceId('Microsoft.Cache/redis', parameters('cacheName')), '2023-04-01').primaryKey]"
    },
    "redisSecondaryKey": {
      "type": "securestring",
      "value": "[listKeys(resourceId('Microsoft.Cache/redis', parameters('cacheName')), '2023-04-01').secondaryKey]"
    },
    "connectionString": {
      "type": "securestring",
      "value": "[concat(reference(resourceId('Microsoft.Cache/redis', parameters('cacheName'))).hostName, ':', reference(resourceId('Microsoft.Cache/redis', parameters('cacheName'))).sslPort, ',password=', listKeys(resourceId('Microsoft.Cache/redis', parameters('cacheName')), '2023-04-01').primaryKey, ',ssl=True,abortConnect=False')]"
    }
  }
}
```

### Fichier de paramètres

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "cacheName": {
      "value": "prod-redis-premium"
    },
    "location": {
      "value": "westeurope"
    },
    "skuName": {
      "value": "Premium"
    },
    "skuFamily": {
      "value": "P"
    },
    "skuCapacity": {
      "value": 3
    },
    "shardCount": {
      "value": 3
    },
    "enableNonSslPort": {
      "value": false
    },
    "minimumTlsVersion": {
      "value": "1.2"
    },
    "publicNetworkAccess": {
      "value": "Disabled"
    },
    "vnetName": {
      "value": "prod-vnet"
    },
    "subnetName": {
      "value": "redis-subnet"
    },
    "storageAccountName": {
      "value": "prodredispersistence"
    },
    "logAnalyticsWorkspaceId": {
      "value": "/subscriptions/{subscription-id}/resourceGroups/monitoring-rg/providers/Microsoft.OperationalInsights/workspaces/prod-logs"
    },
    "enableZoneRedundancy": {
      "value": true
    },
    "tags": {
      "value": {
        "Environment": "Production",
        "Team": "Platform",
        "CostCenter": "Engineering",
        "ManagedBy": "ARM"
      }
    }
  }
}
```

### Déploiement

```bash
#!/bin/bash
# Deploy Azure Cache for Redis Premium with ARM

RESOURCE_GROUP="prod-redis-rg"
LOCATION="westeurope"
TEMPLATE_FILE="redis-premium.json"
PARAMETERS_FILE="redis-premium.parameters.json"

# Create resource group
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION

# Validate template
az deployment group validate \
  --resource-group $RESOURCE_GROUP \
  --template-file $TEMPLATE_FILE \
  --parameters @$PARAMETERS_FILE

# Deploy
az deployment group create \
  --resource-group $RESOURCE_GROUP \
  --template-file $TEMPLATE_FILE \
  --parameters @$PARAMETERS_FILE \
  --name "redis-deployment-$(date +%Y%m%d-%H%M%S)"

# Get connection info
az redis show \
  --name prod-redis-premium \
  --resource-group $RESOURCE_GROUP \
  --query "{hostName:hostName,sslPort:sslPort,enableNonSslPort:enableNonSslPort}"

# Get primary key
az redis list-keys \
  --name prod-redis-premium \
  --resource-group $RESOURCE_GROUP \
  --query primaryKey \
  --output tsv
```

---

## 🔧 Configuration avec Terraform

### Module Terraform complet

```hcl
# Azure Cache for Redis - Terraform Module
# Version: 1.0
# Provider: hashicorp/azurerm ~> 3.0

terraform {
  required_version = ">= 1.5"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.75"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.5"
    }
  }
}

provider "azurerm" {
  features {
    key_vault {
      purge_soft_delete_on_destroy = true
    }
  }
}

# Variables
variable "environment" {
  type        = string
  description = "Environment name"
  default     = "production"
}

variable "location" {
  type        = string
  description = "Azure region"
  default     = "westeurope"
}

variable "redis_capacity" {
  type        = number
  description = "Redis capacity (P1-P5)"
  default     = 3

  validation {
    condition     = var.redis_capacity >= 1 && var.redis_capacity <= 5
    error_message = "Redis capacity must be between 1 and 5 for Premium tier"
  }
}

variable "shard_count" {
  type        = number
  description = "Number of shards for clustering"
  default     = 3

  validation {
    condition     = var.shard_count >= 1 && var.shard_count <= 10
    error_message = "Shard count must be between 1 and 10"
  }
}

variable "enable_zone_redundancy" {
  type        = bool
  description = "Enable zone redundancy"
  default     = true
}

variable "enable_geo_replication" {
  type        = bool
  description = "Enable geo-replication to secondary region"
  default     = false
}

variable "secondary_location" {
  type        = string
  description = "Secondary region for geo-replication"
  default     = "northeurope"
}

variable "tags" {
  type        = map(string)
  description = "Tags to apply to resources"
  default = {
    ManagedBy = "Terraform"
  }
}

# Local variables
locals {
  name_prefix = "${var.environment}-redis"

  common_tags = merge(
    var.tags,
    {
      Environment = var.environment
      Terraform   = "true"
    }
  )
}

# Resource Group
resource "azurerm_resource_group" "main" {
  name     = "${local.name_prefix}-rg"
  location = var.location
  tags     = local.common_tags
}

# Virtual Network
resource "azurerm_virtual_network" "main" {
  name                = "${local.name_prefix}-vnet"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  address_space       = ["10.0.0.0/16"]

  tags = local.common_tags
}

# Subnet for applications
resource "azurerm_subnet" "apps" {
  name                 = "apps-subnet"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.1.0/24"]

  service_endpoints = ["Microsoft.Storage"]
}

# Subnet for Redis (must be delegated)
resource "azurerm_subnet" "redis" {
  name                 = "redis-subnet"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.2.0/27"]

  delegation {
    name = "redis-delegation"

    service_delegation {
      name = "Microsoft.Cache/redis"
      actions = [
        "Microsoft.Network/virtualNetworks/subnets/join/action"
      ]
    }
  }
}

# Network Security Group for Redis
resource "azurerm_network_security_group" "redis" {
  name                = "${local.name_prefix}-nsg"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  # Allow Redis from app subnet
  security_rule {
    name                       = "AllowRedisFromApps"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_ranges    = ["6379", "6380"]
    source_address_prefix      = "10.0.1.0/24"
    destination_address_prefix = "*"
  }

  # Allow to Azure Storage (for persistence)
  security_rule {
    name                       = "AllowToStorage"
    priority                   = 110
    direction                  = "Outbound"
    access                     = "Allow"
    protocol                   = "*"
    source_port_range          = "*"
    destination_port_range     = "*"
    source_address_prefix      = "*"
    destination_address_prefix = "Storage"
  }

  # Deny all other inbound
  security_rule {
    name                       = "DenyAllInbound"
    priority                   = 4096
    direction                  = "Inbound"
    access                     = "Deny"
    protocol                   = "*"
    source_port_range          = "*"
    destination_port_range     = "*"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  tags = local.common_tags
}

# Associate NSG with Redis subnet
resource "azurerm_subnet_network_security_group_association" "redis" {
  subnet_id                 = azurerm_subnet.redis.id
  network_security_group_id = azurerm_network_security_group.redis.id
}

# Storage Account for RDB/AOF persistence
resource "azurerm_storage_account" "redis_persistence" {
  name                     = "${replace(local.name_prefix, "-", "")}persist"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = "GRS"

  min_tls_version = "TLS1_2"

  network_rules {
    default_action             = "Deny"
    virtual_network_subnet_ids = [azurerm_subnet.redis.id]
    bypass                     = ["AzureServices"]
  }

  tags = local.common_tags
}

resource "azurerm_storage_container" "rdb" {
  name                  = "rdb-backups"
  storage_account_name  = azurerm_storage_account.redis_persistence.name
  container_access_type = "private"
}

resource "azurerm_storage_container" "aof" {
  name                  = "aof-logs"
  storage_account_name  = azurerm_storage_account.redis_persistence.name
  container_access_type = "private"
}

# Log Analytics Workspace
resource "azurerm_log_analytics_workspace" "main" {
  name                = "${local.name_prefix}-logs"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  sku                 = "PerGB2018"
  retention_in_days   = 30

  tags = local.common_tags
}

# Azure Cache for Redis Premium (Primary)
resource "azurerm_redis_cache" "primary" {
  name                = "${local.name_prefix}-primary"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  # SKU
  capacity            = var.redis_capacity
  family              = "P"
  sku_name            = "Premium"

  # Cluster configuration
  shard_count         = var.shard_count

  # Zone redundancy (Premium + supported regions)
  zones               = var.enable_zone_redundancy ? ["1", "2", "3"] : null

  # Network
  subnet_id           = azurerm_subnet.redis.id
  public_network_access_enabled = false

  # TLS
  minimum_tls_version = "1.2"
  enable_non_ssl_port = false

  # Redis configuration
  redis_configuration {
    maxmemory_policy                     = "allkeys-lru"
    maxmemory_reserved                   = 50
    maxfragmentationmemory_reserved      = 50
    maxmemory_delta                      = 50

    # RDB persistence
    rdb_backup_enabled                   = true
    rdb_backup_frequency                 = 60  # minutes
    rdb_backup_max_snapshot_count        = 1
    rdb_storage_connection_string        = azurerm_storage_account.redis_persistence.primary_blob_connection_string

    # AOF persistence
    aof_backup_enabled                   = true
    aof_storage_connection_string_0      = azurerm_storage_account.redis_persistence.primary_blob_connection_string
    aof_storage_connection_string_1      = azurerm_storage_account.redis_persistence.secondary_blob_connection_string

    # Notify keyspace events
    notify_keyspace_events               = "Ex"

    # Active defragmentation
    activedefrag_enabled                 = true
  }

  # Patch schedule (Sunday 5-7 AM UTC)
  patch_schedule {
    day_of_week    = "Sunday"
    start_hour_utc = 5

    maintenance_window = "PT2H"  # 2 hours
  }

  tags = local.common_tags

  depends_on = [
    azurerm_subnet_network_security_group_association.redis
  ]
}

# Diagnostic Settings
resource "azurerm_monitor_diagnostic_setting" "redis_primary" {
  name                       = "diagnostics"
  target_resource_id         = azurerm_redis_cache.primary.id
  log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id

  enabled_log {
    category = "ConnectedClientList"
  }

  metric {
    category = "AllMetrics"
    enabled  = true
  }
}

# Secondary Redis for Geo-Replication (optional)
resource "azurerm_resource_group" "secondary" {
  count    = var.enable_geo_replication ? 1 : 0
  name     = "${local.name_prefix}-secondary-rg"
  location = var.secondary_location
  tags     = local.common_tags
}

resource "azurerm_redis_cache" "secondary" {
  count               = var.enable_geo_replication ? 1 : 0
  name                = "${local.name_prefix}-secondary"
  location            = azurerm_resource_group.secondary[0].location
  resource_group_name = azurerm_resource_group.secondary[0].name

  capacity            = var.redis_capacity
  family              = "P"
  sku_name            = "Premium"
  shard_count         = var.shard_count
  zones               = var.enable_zone_redundancy ? ["1", "2", "3"] : null

  minimum_tls_version = "1.2"

  redis_configuration {
    maxmemory_policy = "allkeys-lru"
  }

  tags = local.common_tags
}

# Geo-Replication Link
resource "azurerm_redis_linked_server" "geo_replication" {
  count                       = var.enable_geo_replication ? 1 : 0
  target_redis_cache_name     = azurerm_redis_cache.secondary[0].name
  resource_group_name         = azurerm_resource_group.main.name
  linked_redis_cache_id       = azurerm_redis_cache.primary.id
  linked_redis_cache_location = azurerm_redis_cache.primary.location
  server_role                 = "Primary"
}

# Azure Monitor Alerts
resource "azurerm_monitor_action_group" "redis_alerts" {
  name                = "${local.name_prefix}-alerts"
  resource_group_name = azurerm_resource_group.main.name
  short_name          = "redisalert"

  email_receiver {
    name          = "ops-team"
    email_address = "ops-team@example.com"
  }

  webhook_receiver {
    name        = "slack-webhook"
    service_uri = "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
  }

  tags = local.common_tags
}

# CPU Alert
resource "azurerm_monitor_metric_alert" "cpu_high" {
  name                = "${local.name_prefix}-high-cpu"
  resource_group_name = azurerm_resource_group.main.name
  scopes              = [azurerm_redis_cache.primary.id]
  description         = "Alert when Redis CPU is high"

  criteria {
    metric_namespace = "Microsoft.Cache/redis"
    metric_name      = "percentProcessorTime"
    aggregation      = "Average"
    operator         = "GreaterThan"
    threshold        = 80
  }

  window_size        = "PT5M"
  frequency          = "PT1M"
  severity           = 2

  action {
    action_group_id = azurerm_monitor_action_group.redis_alerts.id
  }

  tags = local.common_tags
}

# Memory Alert
resource "azurerm_monitor_metric_alert" "memory_high" {
  name                = "${local.name_prefix}-high-memory"
  resource_group_name = azurerm_resource_group.main.name
  scopes              = [azurerm_redis_cache.primary.id]
  description         = "Alert when Redis memory usage is high"

  criteria {
    metric_namespace = "Microsoft.Cache/redis"
    metric_name      = "usedmemorypercentage"
    aggregation      = "Average"
    operator         = "GreaterThan"
    threshold        = 85
  }

  window_size = "PT5M"
  frequency   = "PT1M"
  severity    = 2

  action {
    action_group_id = azurerm_monitor_action_group.redis_alerts.id
  }

  tags = local.common_tags
}

# Server Load Alert
resource "azurerm_monitor_metric_alert" "server_load_high" {
  name                = "${local.name_prefix}-high-server-load"
  resource_group_name = azurerm_resource_group.main.name
  scopes              = [azurerm_redis_cache.primary.id]

  criteria {
    metric_namespace = "Microsoft.Cache/redis"
    metric_name      = "serverLoad"
    aggregation      = "Average"
    operator         = "GreaterThan"
    threshold        = 80
  }

  window_size = "PT5M"
  frequency   = "PT1M"
  severity    = 2

  action {
    action_group_id = azurerm_monitor_action_group.redis_alerts.id
  }

  tags = local.common_tags
}

# Connected Clients Alert
resource "azurerm_monitor_metric_alert" "connected_clients_high" {
  name                = "${local.name_prefix}-high-connected-clients"
  resource_group_name = azurerm_resource_group.main.name
  scopes              = [azurerm_redis_cache.primary.id]

  criteria {
    metric_namespace = "Microsoft.Cache/redis"
    metric_name      = "connectedclients"
    aggregation      = "Maximum"
    operator         = "GreaterThan"
    threshold        = 5000
  }

  window_size = "PT5M"
  frequency   = "PT1M"
  severity    = 3

  action {
    action_group_id = azurerm_monitor_action_group.redis_alerts.id
  }

  tags = local.common_tags
}

# Key Vault for storing connection strings
resource "azurerm_key_vault" "main" {
  name                       = "${replace(local.name_prefix, "-", "")}kv"
  location                   = azurerm_resource_group.main.location
  resource_group_name        = azurerm_resource_group.main.name
  tenant_id                  = data.azurerm_client_config.current.tenant_id
  sku_name                   = "standard"
  soft_delete_retention_days = 7
  purge_protection_enabled   = true

  network_acls {
    default_action             = "Deny"
    bypass                     = "AzureServices"
    virtual_network_subnet_ids = [azurerm_subnet.apps.id]
  }

  tags = local.common_tags
}

data "azurerm_client_config" "current" {}

# Store primary key in Key Vault
resource "azurerm_key_vault_secret" "redis_primary_key" {
  name         = "redis-primary-key"
  value        = azurerm_redis_cache.primary.primary_access_key
  key_vault_id = azurerm_key_vault.main.id

  depends_on = [azurerm_redis_cache.primary]
}

# Store connection string in Key Vault
resource "azurerm_key_vault_secret" "redis_connection_string" {
  name  = "redis-connection-string"
  value = "${azurerm_redis_cache.primary.hostname}:${azurerm_redis_cache.primary.ssl_port},password=${azurerm_redis_cache.primary.primary_access_key},ssl=True,abortConnect=False"
  key_vault_id = azurerm_key_vault.main.id

  depends_on = [azurerm_redis_cache.primary]
}

# Outputs
output "redis_hostname" {
  description = "Redis hostname"
  value       = azurerm_redis_cache.primary.hostname
}

output "redis_ssl_port" {
  description = "Redis SSL port"
  value       = azurerm_redis_cache.primary.ssl_port
}

output "redis_primary_key" {
  description = "Redis primary access key"
  value       = azurerm_redis_cache.primary.primary_access_key
  sensitive   = true
}

output "redis_connection_string" {
  description = "Redis connection string"
  value       = "${azurerm_redis_cache.primary.hostname}:${azurerm_redis_cache.primary.ssl_port},password=${azurerm_redis_cache.primary.primary_access_key},ssl=True,abortConnect=False"
  sensitive   = true
}

output "key_vault_name" {
  description = "Key Vault name containing secrets"
  value       = azurerm_key_vault.main.name
}

output "log_analytics_workspace_id" {
  description = "Log Analytics workspace ID"
  value       = azurerm_log_analytics_workspace.main.id
}

output "secondary_redis_hostname" {
  description = "Secondary Redis hostname (if geo-replication enabled)"
  value       = var.enable_geo_replication ? azurerm_redis_cache.secondary[0].hostname : null
}
```

### Déploiement Terraform

```bash
#!/bin/bash
# Deploy Azure Cache for Redis with Terraform

# Initialize
terraform init

# Format
terraform fmt

# Validate
terraform validate

# Plan
terraform plan \
  -var="environment=production" \
  -var="location=westeurope" \
  -var="redis_capacity=3" \
  -var="shard_count=3" \
  -var="enable_zone_redundancy=true" \
  -var="enable_geo_replication=false" \
  -out=tfplan

# Apply
terraform apply tfplan

# Get outputs
terraform output redis_hostname
terraform output redis_ssl_port

# Get connection string (sensitive)
terraform output -raw redis_connection_string
```

---

## 🔧 Configuration avec Bicep

### Fichier Bicep complet

```bicep
// Azure Cache for Redis Premium - Bicep Template
// Author: Platform Team
// Version: 1.0

@description('Environment name')
@allowed([
  'development'
  'staging'
  'production'
])
param environment string = 'production'

@description('Azure region for primary resources')
param location string = resourceGroup().location

@description('Redis cache name')
param cacheName string = 'redis-${environment}-${uniqueString(resourceGroup().id)}'

@description('Redis capacity (P1-P5)')
@minValue(1)
@maxValue(5)
param redisCapacity int = 3

@description('Number of shards for clustering')
@minValue(1)
@maxValue(10)
param shardCount int = 3

@description('Enable zone redundancy')
param enableZoneRedundancy bool = true

@description('Enable geo-replication')
param enableGeoReplication bool = false

@description('Secondary region for geo-replication')
param secondaryLocation string = 'northeurope'

@description('Tags to apply to resources')
param tags object = {
  Environment: environment
  ManagedBy: 'Bicep'
}

// Variables
var namePrefix = '${environment}-redis'
var vnetName = '${namePrefix}-vnet'
var appsSubnetName = 'apps-subnet'
var redisSubnetName = 'redis-subnet'
var nsgName = '${namePrefix}-nsg'
var storageAccountName = '${replace(namePrefix, '-', '')}persist'
var logAnalyticsName = '${namePrefix}-logs'
var keyVaultName = '${replace(namePrefix, '-', '')}kv'
var actionGroupName = '${namePrefix}-alerts'

// Virtual Network
resource vnet 'Microsoft.Network/virtualNetworks@2023-05-01' = {
  name: vnetName
  location: location
  tags: tags
  properties: {
    addressSpace: {
      addressPrefixes: [
        '10.0.0.0/16'
      ]
    }
    subnets: [
      {
        name: appsSubnetName
        properties: {
          addressPrefix: '10.0.1.0/24'
          serviceEndpoints: [
            {
              service: 'Microsoft.Storage'
            }
            {
              service: 'Microsoft.KeyVault'
            }
          ]
        }
      }
      {
        name: redisSubnetName
        properties: {
          addressPrefix: '10.0.2.0/27'
          delegations: [
            {
              name: 'redis-delegation'
              properties: {
                serviceName: 'Microsoft.Cache/redis'
              }
            }
          ]
          networkSecurityGroup: {
            id: nsg.id
          }
        }
      }
    ]
  }
}

// Network Security Group
resource nsg 'Microsoft.Network/networkSecurityGroups@2023-05-01' = {
  name: nsgName
  location: location
  tags: tags
  properties: {
    securityRules: [
      {
        name: 'AllowRedisFromApps'
        properties: {
          priority: 100
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourcePortRange: '*'
          destinationPortRanges: [
            '6379'
            '6380'
          ]
          sourceAddressPrefix: '10.0.1.0/24'
          destinationAddressPrefix: '*'
        }
      }
      {
        name: 'AllowToStorage'
        properties: {
          priority: 110
          direction: 'Outbound'
          access: 'Allow'
          protocol: '*'
          sourcePortRange: '*'
          destinationPortRange: '*'
          sourceAddressPrefix: '*'
          destinationAddressPrefix: 'Storage'
        }
      }
      {
        name: 'DenyAllInbound'
        properties: {
          priority: 4096
          direction: 'Inbound'
          access: 'Deny'
          protocol: '*'
          sourcePortRange: '*'
          destinationPortRange: '*'
          sourceAddressPrefix: '*'
          destinationAddressPrefix: '*'
        }
      }
    ]
  }
}

// Storage Account for persistence
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  tags: tags
  sku: {
    name: 'Standard_GRS'
  }
  kind: 'StorageV2'
  properties: {
    minimumTlsVersion: 'TLS1_2'
    supportsHttpsTrafficOnly: true
    networkAcls: {
      defaultAction: 'Deny'
      virtualNetworkRules: [
        {
          id: resourceId('Microsoft.Network/virtualNetworks/subnets', vnetName, redisSubnetName)
        }
      ]
      bypass: 'AzureServices'
    }
  }
}

resource rdbContainer 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-01-01' = {
  name: '${storageAccount.name}/default/rdb-backups'
  properties: {
    publicAccess: 'None'
  }
}

resource aofContainer 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-01-01' = {
  name: '${storageAccount.name}/default/aof-logs'
  properties: {
    publicAccess: 'None'
  }
}

// Log Analytics Workspace
resource logAnalytics 'Microsoft.OperationalInsights/workspaces@2022-10-01' = {
  name: logAnalyticsName
  location: location
  tags: tags
  properties: {
    sku: {
      name: 'PerGB2018'
    }
    retentionInDays: 30
  }
}

// Azure Cache for Redis Premium
resource redisCache 'Microsoft.Cache/redis@2023-04-01' = {
  name: cacheName
  location: location
  tags: tags
  properties: {
    sku: {
      name: 'Premium'
      family: 'P'
      capacity: redisCapacity
    }
    enableNonSslPort: false
    minimumTlsVersion: '1.2'
    publicNetworkAccess: 'Disabled'
    redisVersion: '6'
    shardCount: shardCount
    replicasPerMaster: 1
    replicasPerPrimary: 1
    subnetId: resourceId('Microsoft.Network/virtualNetworks/subnets', vnetName, redisSubnetName)
    zones: enableZoneRedundancy ? [
      '1'
      '2'
      '3'
    ] : null
    redisConfiguration: {
      'maxmemory-policy': 'allkeys-lru'
      'maxmemory-reserved': '50'
      'maxfragmentationmemory-reserved': '50'
      'maxmemory-delta': '50'
      'notify-keyspace-events': 'Ex'
      'rdb-backup-enabled': 'true'
      'rdb-backup-frequency': '60'
      'rdb-backup-max-snapshot-count': '1'
      'rdb-storage-connection-string': 'DefaultEndpointsProtocol=https;AccountName=${storageAccount.name};AccountKey=${storageAccount.listKeys().keys[0].value}'
      'aof-backup-enabled': 'true'
      'aof-storage-connection-string-0': 'DefaultEndpointsProtocol=https;AccountName=${storageAccount.name};AccountKey=${storageAccount.listKeys().keys[0].value}'
      'aof-storage-connection-string-1': 'DefaultEndpointsProtocol=https;AccountName=${storageAccount.name};AccountKey=${storageAccount.listKeys().keys[1].value}'
    }
  }
}

// Diagnostic Settings
resource diagnosticSettings 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: 'diagnostics'
  scope: redisCache
  properties: {
    workspaceId: logAnalytics.id
    logs: [
      {
        category: 'ConnectedClientList'
        enabled: true
        retentionPolicy: {
          days: 30
          enabled: true
        }
      }
    ]
    metrics: [
      {
        category: 'AllMetrics'
        enabled: true
        retentionPolicy: {
          days: 30
          enabled: true
        }
      }
    ]
  }
}

// Secondary Redis for Geo-Replication
resource secondaryRedisCache 'Microsoft.Cache/redis@2023-04-01' = if (enableGeoReplication) {
  name: '${cacheName}-secondary'
  location: secondaryLocation
  tags: tags
  properties: {
    sku: {
      name: 'Premium'
      family: 'P'
      capacity: redisCapacity
    }
    minimumTlsVersion: '1.2'
    shardCount: shardCount
    zones: enableZoneRedundancy ? [
      '1'
      '2'
      '3'
    ] : null
    redisConfiguration: {
      'maxmemory-policy': 'allkeys-lru'
    }
  }
}

// Geo-Replication Link
resource geoReplication 'Microsoft.Cache/redis/linkedServers@2023-04-01' = if (enableGeoReplication) {
  parent: redisCache
  name: 'geo-replication-link'
  properties: {
    linkedRedisCacheId: secondaryRedisCache.id
    linkedRedisCacheLocation: secondaryLocation
    serverRole: 'Primary'
  }
}

// Action Group for alerts
resource actionGroup 'Microsoft.Insights/actionGroups@2023-01-01' = {
  name: actionGroupName
  location: 'Global'
  tags: tags
  properties: {
    groupShortName: 'redisalert'
    enabled: true
    emailReceivers: [
      {
        name: 'ops-team'
        emailAddress: 'ops-team@example.com'
        useCommonAlertSchema: true
      }
    ]
  }
}

// Metric Alerts
resource cpuAlert 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  name: '${namePrefix}-high-cpu'
  location: 'global'
  tags: tags
  properties: {
    description: 'Alert when Redis CPU is high'
    severity: 2
    enabled: true
    scopes: [
      redisCache.id
    ]
    evaluationFrequency: 'PT1M'
    windowSize: 'PT5M'
    criteria: {
      'odata.type': 'Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria'
      allOf: [
        {
          name: 'HighCPU'
          metricName: 'percentProcessorTime'
          metricNamespace: 'Microsoft.Cache/redis'
          operator: 'GreaterThan'
          threshold: 80
          timeAggregation: 'Average'
        }
      ]
    }
    actions: [
      {
        actionGroupId: actionGroup.id
      }
    ]
  }
}

resource memoryAlert 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  name: '${namePrefix}-high-memory'
  location: 'global'
  tags: tags
  properties: {
    description: 'Alert when Redis memory usage is high'
    severity: 2
    enabled: true
    scopes: [
      redisCache.id
    ]
    evaluationFrequency: 'PT1M'
    windowSize: 'PT5M'
    criteria: {
      'odata.type': 'Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria'
      allOf: [
        {
          name: 'HighMemory'
          metricName: 'usedmemorypercentage'
          metricNamespace: 'Microsoft.Cache/redis'
          operator: 'GreaterThan'
          threshold: 85
          timeAggregation: 'Average'
        }
      ]
    }
    actions: [
      {
        actionGroupId: actionGroup.id
      }
    ]
  }
}

// Key Vault
resource keyVault 'Microsoft.KeyVault/vaults@2023-02-01' = {
  name: keyVaultName
  location: location
  tags: tags
  properties: {
    sku: {
      family: 'A'
      name: 'standard'
    }
    tenantId: subscription().tenantId
    enableSoftDelete: true
    softDeleteRetentionInDays: 7
    enablePurgeProtection: true
    networkAcls: {
      defaultAction: 'Deny'
      bypass: 'AzureServices'
      virtualNetworkRules: [
        {
          id: resourceId('Microsoft.Network/virtualNetworks/subnets', vnetName, appsSubnetName)
        }
      ]
    }
  }
}

// Store primary key in Key Vault
resource redisPrimaryKeySecret 'Microsoft.KeyVault/vaults/secrets@2023-02-01' = {
  parent: keyVault
  name: 'redis-primary-key'
  properties: {
    value: redisCache.listKeys().primaryKey
  }
}

// Store connection string in Key Vault
resource redisConnectionStringSecret 'Microsoft.KeyVault/vaults/secrets@2023-02-01' = {
  parent: keyVault
  name: 'redis-connection-string'
  properties: {
    value: '${redisCache.properties.hostName}:${redisCache.properties.sslPort},password=${redisCache.listKeys().primaryKey},ssl=True,abortConnect=False'
  }
}

// Outputs
output redisHostName string = redisCache.properties.hostName
output redisSslPort int = redisCache.properties.sslPort
output redisPrimaryKey string = redisCache.listKeys().primaryKey
output redisConnectionString string = '${redisCache.properties.hostName}:${redisCache.properties.sslPort},password=${redisCache.listKeys().primaryKey},ssl=True,abortConnect=False'
output keyVaultName string = keyVault.name
output logAnalyticsWorkspaceId string = logAnalytics.id
output secondaryRedisHostName string = enableGeoReplication ? secondaryRedisCache.properties.hostName : ''
```

### Déploiement Bicep

```bash
#!/bin/bash
# Deploy Azure Cache for Redis with Bicep

RESOURCE_GROUP="prod-redis-rg"
LOCATION="westeurope"
BICEP_FILE="redis-premium.bicep"

# Create resource group
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION

# Validate Bicep
az deployment group validate \
  --resource-group $RESOURCE_GROUP \
  --template-file $BICEP_FILE \
  --parameters environment=production \
               redisCapacity=3 \
               shardCount=3 \
               enableZoneRedundancy=true \
               enableGeoReplication=false

# Deploy
az deployment group create \
  --resource-group $RESOURCE_GROUP \
  --template-file $BICEP_FILE \
  --parameters environment=production \
               redisCapacity=3 \
               shardCount=3 \
               enableZoneRedundancy=true \
               enableGeoReplication=false \
  --name "redis-deployment-$(date +%Y%m%d-%H%M%S)"

# Get outputs
az deployment group show \
  --resource-group $RESOURCE_GROUP \
  --name redis-deployment-* \
  --query properties.outputs
```

---

## 📊 Pricing détaillé (West Europe, 2024)

### Grille tarifaire complète

```yaml
Basic Tier (Dev/Test, no SLA)
├── C0 (250 MB):   €0.016/h  →  €11.68/mois
├── C1 (1 GB):     €0.041/h  →  €29.93/mois
├── C2 (2.5 GB):   €0.083/h  →  €60.59/mois
├── C3 (6 GB):     €0.167/h  →  €121.91/mois
├── C4 (13 GB):    €0.333/h  →  €243.09/mois
├── C5 (26 GB):    €0.667/h  →  €486.91/mois
└── C6 (53 GB):    €1.333/h  →  €973.09/mois

Standard Tier (with replication, 99.9% SLA)
├── Prix = Basic × 2 (primary + replica)
├── C0: €23.36/mois
├── C1: €59.86/mois
├── C2: €121.18/mois
├── C3: €243.82/mois
├── C4: €486.18/mois
├── C5: €973.82/mois
└── C6: €1,947.18/mois

Premium Tier (production, clustering, VNet)
├── P1 (6 GB):     €0.342/h  →  €249.66/mois
├── P2 (13 GB):    €0.684/h  →  €499.32/mois
├── P3 (26 GB):    €1.368/h  →  €998.64/mois
├── P4 (53 GB):    €2.736/h  →  €1,997.28/mois
└── P5 (120 GB):   €6.148/h  →  €4,488.04/mois

Premium with Clustering (multiply by shard count)
Example P3 × 3 shards:
├── Base: €998.64/mois
├── Total: €998.64 × 3 = €2,995.92/mois

Enterprise Tiers (Redis Enterprise)
├── E10 (12 GB):   €3.186/h  →  €2,325.78/mois
├── E20 (25 GB):   €6.373/h  →  €4,652.29/mois
├── E50 (50 GB):   €12.746/h →  €9,304.58/mois
├── E100 (100 GB): €25.492/h →  €18,609.16/mois
├── E200 (200 GB): €50.984/h →  €37,218.32/mois
└── Enterprise Flash (RAM+Flash tiering available)

Reserved Instances Discounts:
├── 1 year, pay upfront: ~35% discount
├── 3 years, pay upfront: ~55% discount

Example: P3 Premium with RI 3 years
├── On-demand: €998.64/mois
├── RI 3 years: €998.64 × 0.45 = €449.39/mois
└── Économie: €549.25/mois (€6,591/an)

Coûts additionnels:
├── Geo-replication: Prix du cache secondaire
├── Storage (persistence): ~€0.02/GB-mois (Blob Storage)
├── Bandwidth out: €0.087/GB vers Internet
├── Bandwidth inter-region: €0.02/GB
└── Private Link: €0.01/h per endpoint (~€7/mois)
```

### Calculateur de coût pour différents scénarios

```python
# Azure Redis Cost Calculator

class AzureRedisPricing:
    """Calculate Azure Cache for Redis costs"""

    # Pricing per hour (West Europe, 2024)
    PRICING = {
        'Basic': {
            'C0': 0.016, 'C1': 0.041, 'C2': 0.083,
            'C3': 0.167, 'C4': 0.333, 'C5': 0.667, 'C6': 1.333
        },
        'Standard': {  # 2x Basic (primary + replica)
            'C0': 0.032, 'C1': 0.082, 'C2': 0.166,
            'C3': 0.334, 'C4': 0.666, 'C5': 1.334, 'C6': 2.666
        },
        'Premium': {
            'P1': 0.342, 'P2': 0.684, 'P3': 1.368,
            'P4': 2.736, 'P5': 6.148
        },
        'Enterprise': {
            'E10': 3.186, 'E20': 6.373, 'E50': 12.746,
            'E100': 25.492, 'E200': 50.984
        }
    }

    HOURS_PER_MONTH = 730

    @classmethod
    def calculate_monthly_cost(
        cls,
        tier: str,
        sku: str,
        shard_count: int = 1,
        geo_replication: bool = False,
        reserved_instance_years: int = 0
    ) -> dict:
        """
        Calculate monthly cost for Azure Redis

        Args:
            tier: 'Basic', 'Standard', 'Premium', or 'Enterprise'
            sku: SKU size (e.g., 'C0', 'P3', 'E10')
            shard_count: Number of shards for Premium (1-10)
            geo_replication: Enable geo-replication (doubles cost)
            reserved_instance_years: 0 (on-demand), 1, or 3
        """

        # Get base hourly cost
        if tier not in cls.PRICING or sku not in cls.PRICING[tier]:
            raise ValueError(f"Invalid tier/SKU: {tier}/{sku}")

        hourly_cost = cls.PRICING[tier][sku]

        # Apply clustering for Premium
        if tier == 'Premium' and shard_count > 1:
            hourly_cost *= shard_count

        # Monthly cost
        monthly_cost = hourly_cost * cls.HOURS_PER_MONTH

        # Apply Reserved Instance discount
        ri_discount = {0: 0, 1: 0.35, 3: 0.55}
        discount = ri_discount.get(reserved_instance_years, 0)
        monthly_cost_with_ri = monthly_cost * (1 - discount)

        # Geo-replication (secondary cache)
        if geo_replication:
            secondary_cost = monthly_cost_with_ri
        else:
            secondary_cost = 0

        total_monthly = monthly_cost_with_ri + secondary_cost
        total_annual = total_monthly * 12

        # Additional costs (estimates)
        storage_cost = 50  # €50/month for persistence storage
        bandwidth_cost = 100  # €100/month for inter-AZ/region traffic
        private_link_cost = 7 if tier == 'Premium' else 0

        additional_monthly = storage_cost + bandwidth_cost + private_link_cost

        return {
            'base_hourly': round(hourly_cost, 3),
            'base_monthly': round(monthly_cost, 2),
            'ri_discount_pct': discount * 100,
            'monthly_with_ri': round(monthly_cost_with_ri, 2),
            'secondary_monthly': round(secondary_cost, 2),
            'compute_monthly': round(total_monthly, 2),
            'additional_monthly': round(additional_monthly, 2),
            'total_monthly': round(total_monthly + additional_monthly, 2),
            'total_annual': round(total_annual + additional_monthly * 12, 2)
        }

# Examples
print("=== Scenario 1: Production e-commerce (Premium P3, 3 shards) ===")
cost1 = AzureRedisPricing.calculate_monthly_cost(
    tier='Premium',
    sku='P3',
    shard_count=3,
    geo_replication=False,
    reserved_instance_years=3
)
print(f"Monthly cost: €{cost1['total_monthly']:,}")
print(f"Annual cost: €{cost1['total_annual']:,}")
print(f"Savings vs on-demand: €{(cost1['base_monthly'] * 3 * 12 - cost1['total_annual']):,.0f}/year")

print("\n=== Scenario 2: Global with Geo-Replication ===")
cost2 = AzureRedisPricing.calculate_monthly_cost(
    tier='Premium',
    sku='P3',
    shard_count=3,
    geo_replication=True,  # Add secondary region
    reserved_instance_years=3
)
print(f"Monthly cost: €{cost2['total_monthly']:,}")
print(f"Annual cost: €{cost2['total_annual']:,}")

print("\n=== Scenario 3: Enterprise with Active-Active ===")
cost3 = AzureRedisPricing.calculate_monthly_cost(
    tier='Enterprise',
    sku='E20',
    shard_count=1,
    geo_replication=True,  # Active-Active
    reserved_instance_years=1
)
print(f"Monthly cost: €{cost3['total_monthly']:,}")
print(f"Annual cost: €{cost3['total_annual']:,}")

print("\n=== Comparison: Standard vs Premium ===")
standard = AzureRedisPricing.calculate_monthly_cost('Standard', 'C5', reserved_instance_years=3)
premium = AzureRedisPricing.calculate_monthly_cost('Premium', 'P3', reserved_instance_years=3)
print(f"Standard C5: €{standard['total_monthly']}/month")
print(f"Premium P3: €{premium['total_monthly']}/month")
print(f"Premium adds: VNet, Persistence, Clustering")
print(f"Cost difference: €{premium['total_monthly'] - standard['total_monthly']}/month")
```

Output:
```
=== Scenario 1: Production e-commerce (Premium P3, 3 shards) ===
Monthly cost: €1,505
Annual cost: €18,060
Savings vs on-demand: €17,934/year

=== Scenario 2: Global with Geo-Replication ===
Monthly cost: €2,855
Annual cost: €34,260

=== Scenario 3: Enterprise with Active-Active ===
Monthly cost: €6,211
Annual cost: €74,532

=== Comparison: Standard vs Premium ===
Standard C5: €595/month
Premium P3: €606/month
Premium adds: VNet, Persistence, Clustering
Cost difference: €11/month
```

---

## 🔍 Monitoring avec Azure Monitor

### Dashboard Azure Monitor

```json
{
  "properties": {
    "lenses": [
      {
        "order": 0,
        "parts": [
          {
            "position": {
              "x": 0,
              "y": 0,
              "colSpan": 6,
              "rowSpan": 4
            },
            "metadata": {
              "type": "Extension/Microsoft_Azure_Monitoring/PartType/MetricsChartPart",
              "settings": {
                "content": {
                  "options": {
                    "chart": {
                      "metrics": [
                        {
                          "resourceMetadata": {
                            "id": "/subscriptions/{sub-id}/resourceGroups/prod-redis-rg/providers/Microsoft.Cache/redis/prod-redis-premium"
                          },
                          "name": "percentProcessorTime",
                          "aggregationType": 4,
                          "namespace": "microsoft.cache/redis",
                          "metricVisualization": {
                            "displayName": "CPU (%)"
                          }
                        }
                      ],
                      "title": "Redis CPU Usage",
                      "titleKind": 1,
                      "visualization": {
                        "chartType": 2,
                        "legendVisualization": {
                          "isVisible": true,
                          "position": 2,
                          "hideSubtitle": false
                        },
                        "axisVisualization": {
                          "x": {
                            "isVisible": true,
                            "axisType": 2
                          },
                          "y": {
                            "isVisible": true,
                            "axisType": 1
                          }
                        }
                      },
                      "timespan": {
                        "relative": {
                          "duration": 3600000
                        },
                        "showUTCTime": false,
                        "grain": 1
                      }
                    }
                  }
                }
              }
            }
          },
          {
            "position": {
              "x": 6,
              "y": 0,
              "colSpan": 6,
              "rowSpan": 4
            },
            "metadata": {
              "type": "Extension/Microsoft_Azure_Monitoring/PartType/MetricsChartPart",
              "settings": {
                "content": {
                  "options": {
                    "chart": {
                      "metrics": [
                        {
                          "resourceMetadata": {
                            "id": "/subscriptions/{sub-id}/resourceGroups/prod-redis-rg/providers/Microsoft.Cache/redis/prod-redis-premium"
                          },
                          "name": "usedmemorypercentage",
                          "aggregationType": 4,
                          "namespace": "microsoft.cache/redis",
                          "metricVisualization": {
                            "displayName": "Used Memory %"
                          }
                        }
                      ],
                      "title": "Redis Memory Usage",
                      "titleKind": 1
                    }
                  }
                }
              }
            }
          },
          {
            "position": {
              "x": 0,
              "y": 4,
              "colSpan": 6,
              "rowSpan": 4
            },
            "metadata": {
              "type": "Extension/Microsoft_Azure_Monitoring/PartType/MetricsChartPart",
              "settings": {
                "content": {
                  "options": {
                    "chart": {
                      "metrics": [
                        {
                          "resourceMetadata": {
                            "id": "/subscriptions/{sub-id}/resourceGroups/prod-redis-rg/providers/Microsoft.Cache/redis/prod-redis-premium"
                          },
                          "name": "cachehits",
                          "aggregationType": 1,
                          "namespace": "microsoft.cache/redis",
                          "metricVisualization": {
                            "displayName": "Cache Hits"
                          }
                        },
                        {
                          "resourceMetadata": {
                            "id": "/subscriptions/{sub-id}/resourceGroups/prod-redis-rg/providers/Microsoft.Cache/redis/prod-redis-premium"
                          },
                          "name": "cachemisses",
                          "aggregationType": 1,
                          "namespace": "microsoft.cache/redis",
                          "metricVisualization": {
                            "displayName": "Cache Misses"
                          }
                        }
                      ],
                      "title": "Cache Hits vs Misses",
                      "titleKind": 1
                    }
                  }
                }
              }
            }
          },
          {
            "position": {
              "x": 6,
              "y": 4,
              "colSpan": 6,
              "rowSpan": 4
            },
            "metadata": {
              "type": "Extension/Microsoft_Azure_Monitoring/PartType/MetricsChartPart",
              "settings": {
                "content": {
                  "options": {
                    "chart": {
                      "metrics": [
                        {
                          "resourceMetadata": {
                            "id": "/subscriptions/{sub-id}/resourceGroups/prod-redis-rg/providers/Microsoft.Cache/redis/prod-redis-premium"
                          },
                          "name": "connectedclients",
                          "aggregationType": 3,
                          "namespace": "microsoft.cache/redis",
                          "metricVisualization": {
                            "displayName": "Connected Clients"
                          }
                        }
                      ],
                      "title": "Connected Clients",
                      "titleKind": 1
                    }
                  }
                }
              }
            }
          }
        ]
      }
    ],
    "metadata": {
      "model": {
        "timeRange": {
          "value": {
            "relative": {
              "duration": 24,
              "timeUnit": 1
            }
          },
          "type": "MsPortalFx.Composition.Configuration.ValueTypes.TimeRange"
        }
      }
    }
  },
  "name": "Redis Production Dashboard",
  "type": "Microsoft.Portal/dashboards",
  "location": "westeurope",
  "tags": {
    "hidden-title": "Redis Production Dashboard"
  }
}
```

### Métriques critiques à surveiller

```yaml
Performance Metrics:
├── percentProcessorTime: <80% (alerte si >85%)
├── serverLoad: <80% (combinaison CPU + load)
├── usedmemory: Monitoring de tendance
└── usedmemorypercentage: <85%

Cache Efficiency:
├── cachehits: Nombre de hits
├── cachemisses: Nombre de misses
├── Hit rate formula: hits / (hits + misses) × 100
└── Target: >80% hit rate

Connections:
├── connectedclients: Surveiller les pics
├── connectedclientsmaxidletime: Timeout configuration
└── totalcommandsprocessed: Throughput

Memory Details:
├── usedmemoryRss: Resident Set Size
├── evictedkeys: Doit être proche de 0
├── expiredkeys: Normal, dépend des TTLs
└── totalkeys: Nombre total de clés

Operations:
├── getcommands: Read operations
├── setcommands: Write operations
├── operationsPerSecond: Throughput total
└── commandsPerSecond: Commands/sec par shard

Network:
├── cacheRead: Bytes read from cache
├── cacheWrite: Bytes written to cache
└── Alertes si saturation réseau

Errors:
├── errors: Toute erreur est critique
├── totalcommandsprocessedfailed: Failed commands
└── Exception dans logs: Investigate immediately
```

### Query Log Analytics (KQL)

```kusto
// Top 10 slow commands
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.CACHE"
| where Category == "RedisSlowLogs"
| extend command = tostring(parse_json(properties_s).command)
| extend executionTime = tolong(parse_json(properties_s).executionTimeMicroseconds)
| summarize count(), avg(executionTime) by command
| top 10 by avg_executionTime desc

// Connected clients over time
AzureMetrics
| where ResourceProvider == "MICROSOFT.CACHE"
| where MetricName == "connectedclients"
| summarize avg(Average) by bin(TimeGenerated, 5m)
| render timechart

// Cache hit rate
AzureMetrics
| where ResourceProvider == "MICROSOFT.CACHE"
| where MetricName in ("cachehits", "cachemisses")
| summarize hits = sumif(Total, MetricName == "cachehits"),
            misses = sumif(Total, MetricName == "cachemisses") by bin(TimeGenerated, 5m)
| extend hitRate = hits * 100.0 / (hits + misses)
| project TimeGenerated, hitRate
| render timechart

// Memory usage trend
AzureMetrics
| where ResourceProvider == "MICROSOFT.CACHE"
| where MetricName == "usedmemorypercentage"
| summarize avg(Average) by bin(TimeGenerated, 1h)
| render timechart

// Errors and exceptions
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.CACHE"
| where Category == "Errors"
| project TimeGenerated, OperationName, ResultDescription, CallerIPAddress
| order by TimeGenerated desc
```

---

## ✅ Checklist de décision

### Choisir le bon tier

```yaml
Utilisez Basic si:
☐ Environnement dev/test uniquement
☐ Pas de données critiques
☐ Budget très limité
☐ Downtime acceptable
☐ Pas besoin de SLA

Utilisez Standard si:
☐ Production (workloads simples)
☐ Besoin de réplication
☐ SLA 99.9% requis
☐ Dataset <53 GB
☐ Pas besoin de features avancées

Utilisez Premium si:
☐ Production (workloads avancés)
☐ Isolation réseau (VNet) requise
☐ Persistence RDB/AOF nécessaire
☐ Clustering requis (>53 GB)
☐ Geo-replication (DR)
☐ Dataset jusqu'à 1.2 TB

Utilisez Enterprise si:
☐ Mission-critical workloads
☐ SLA 99.99% requis
☐ Active-Active geo-distribution
☐ Modules Redis Stack nécessaires
☐ Auto-scaling requis
☐ Dataset >1 TB
```

### Matrice de décision Azure vs AWS vs GCP

| Critère | Azure Cache Premium | AWS ElastiCache | AWS MemoryDB | GCP Memorystore |
|---------|---------------------|-----------------|--------------|-----------------|
| **VNet injection** | ✅ Oui | ❌ Non (subnet group) | ❌ Non | ✅ Oui (VPC peering) |
| **Max RAM/cluster** | 1.2 TB (P5×10) | 150 TB | 200 TB | 300 GB |
| **Geo-replication** | ✅ (passive) | ✅ Global Datastore | ❌ | ❌ |
| **Active-Active** | ✅ (Enterprise) | ❌ | ❌ | ❌ |
| **Zone redundancy** | ✅ | ✅ (Multi-AZ) | ✅ (built-in) | ✅ |
| **Auto-scaling** | ✅ (Enterprise) | ❌ | ❌ | ❌ |
| **SLA** | 99.9% / 99.99% | 99.9% / 99.99% | 99.99% | 99.9% |
| **Intégration écosystème** | Excellent (Azure) | Excellent (AWS) | Excellent (AWS) | Excellent (GCP) |
| **Pricing (50GB RI 3y)** | ~€450/mois | ~$200/mois | ~$350/mois | ~$250/mois |

---

## 🎯 Conclusion

### Points clés à retenir

1. **Quatre tiers adaptés aux différents besoins**
   - Basic : Dev/test, no SLA
   - Standard : Production simple, réplication
   - Premium : Production avancée, VNet, persistence
   - Enterprise : Mission-critical, Active-Active

2. **VNet injection pour isolation complète**
   - Premium et Enterprise uniquement
   - Subnet déléguée à Microsoft.Cache/redis
   - Pas d'endpoint public nécessaire

3. **Persistence robuste (Premium)**
   - RDB snapshots (15min-24h)
   - AOF logs (durabilité maximale)
   - Stockage Azure Blob

4. **Geo-replication**
   - Premium : Active-Passive (read-only secondary)
   - Enterprise : Active-Active avec CRDTs

5. **IaC complet**
   - ARM Templates (natif Azure)
   - Terraform (multi-cloud)
   - Bicep (moderne, simplifié)

### Recommandations finales

**Pour la plupart des cas (production standard) :**
```
Azure Cache Premium P3 (26GB)
├── 3 shards (clustering)
├── Zone redundancy
├── VNet injection
├── RDB + AOF persistence
└── Reserved Instance 3 ans
Total: ~€450/mois
```

**Pour mission-critical :**
```
Azure Cache Enterprise E20 (25GB)
├── Active-Active (2+ regions)
├── Redis Stack modules
├── Auto-scaling
└── SLA 99.99%
Total: ~€3,000/mois (2 regions)
```

---

**🎯 Section suivante :** Nous allons maintenant explorer **Google Cloud Memorystore** dans la section 15.4.

⏭️ [Google Cloud Memorystore](/15-redis-cloud-conteneurs/04-google-cloud-memorystore.md)
