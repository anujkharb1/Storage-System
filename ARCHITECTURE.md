# VAULT - Distributed Object Storage Architecture

## System Overview

VAULT is a fault-tolerant distributed object storage system designed to handle file storage with automatic replication, data integrity verification, and self-healing capabilities.

## Components

### Storage Engine
- Files are stored in a hierarchical directory structure under each virtual storage node
- Path format: `{node-name}/{prefix}/{sub-prefix}/{object-id}/{filename}`
- SHA-256 checksums computed at upload time for integrity verification

### Replication Strategy
- Objects are replicated to N nodes (configurable replication factor)
- Nodes selected by least-used-space-first algorithm
- First replica designated as "primary"
- All replicas independently verifiable via checksum

### Background Workers
1. **Health Monitor** - Checks node liveness every 10s
2. **Repair Worker** - Scans for under-replicated objects every 30s
3. **Rebalance Worker** - Redistributes objects every 60s
4. **Integrity Worker** - Verifies checksums every 120s

### Data Flow

```
Upload:
  Client → Express → Multer → Storage Engine → N nodes
  ↓
  Prisma → PostgreSQL (metadata)
  ↓
  Socket.IO → Dashboard

Download:
  Client → Express → Find healthy replica → Storage Engine → Verify checksum → Respond

Delete:
  Client → Express → Remove from all nodes → Update DB → Invalidate cache
```

### Failure Handling
1. **Node Failure**: Detected by health monitor → replicas on failed node marked MISSING → repair worker creates new replicas on healthy nodes
2. **Data Corruption**: Detected by integrity check → replica marked CORRUPTED → repair worker copies from healthy replica
3. **Network Partition**: Multiple nodes isolated → affected objects marked DEGRADED → auto-repair after recovery

## Database Schema

See `backend/prisma/schema.prisma` for the complete data model.

## API Design

All APIs follow RESTful conventions with JSON responses:
```json
{
  "success": true,
  "data": { ... }
}
```

Error responses:
```json
{
  "success": false,
  "error": {
    "message": "Error description",
    "statusCode": 404
  }
}
```
