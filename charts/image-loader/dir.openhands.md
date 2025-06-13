# Image Loader

K8s infrastructure for preloading container images and maintaining warm node pool. Reduces startup latency for OpenHands runtime environments.

## Components & Configuration
- **DaemonSet**: Preloads `ghcr.io/all-hands-ai/runtime` on nodes with `sysbox-install: "yes"` label (100m CPU/128Mi)
- **Node Overprovisioner**: Maintains warm node pool (10 replicas in production, 2500m CPU/7500Mi)
- **Priority Class**: Ensures proper scheduling (-10 priority)
- **Runtime**: `sysbox-runc` (migrated from gVisor Feb 2025)
- **Update Strategy**: 100% simultaneous pod updates

## Directory Structure
- **chart/**: Helm chart templates and values
- **envs/**: Environment-specific configurations

## Evolution
- Created Oct 2024
- Feb 2025: Migrated from gVisor to sysbox-runc runtime
- Mar 2025: Increased production replicas from 5 to 10
- Mar 2025: Implemented 100% simultaneous pod updates

Stable component with consistent performance. Changes primarily focus on scaling and optimization rather than bug fixes.