# MongoDB without AVX (mongodb-no-avx)

Docker image to build MongoDB from source without AVX CPU instruction requirements. This is designed for use with [mbentley/docker-omada-controller](https://github.com/mbentley/docker-omada-controller) on older CPUs that don't support AVX instructions.

Based on the work from [alanedwardes/mongodb-without-avx](https://github.com/alanedwardes/mongodb-without-avx), updated for MongoDB 8.x with Bazel build system.

## Pre-built Images

Pre-built images are available from GitHub Container Registry:

```bash
docker pull ghcr.io/fenio/mongodb-no-avx:8.0.17  # LTS (recommended)
docker pull ghcr.io/fenio/mongodb-no-avx:8.2.3   # Latest
docker pull ghcr.io/fenio/mongodb-no-avx:latest  # Same as 8.0.17
```

## Why?

MongoDB 5.0+ requires AVX (Advanced Vector Extensions) CPU instructions by default. Many older CPUs (pre-2011) and some virtualization platforms don't support AVX. This project builds MongoDB from source with AVX optimizations disabled.

### Technical Details

MongoDB 8.x uses `-march=sandybridge` compiler flag which requires AVX support. This build replaces it with `-march=x86-64-v2` which:
- Supports SSE4.2 and POPCNT (reasonable baseline for modern CPUs)
- Does NOT require AVX instructions
- Works on Intel Westmere (2010) and newer, AMD Bulldozer (2011) and newer

## Building the Image

### Quick Build

```bash
docker build -t mongodb-no-avx:8.0.17 .
```

### Build with Specific Version

```bash
docker build \
  --build-arg MONGO_VERSION=8.0.17 \
  --build-arg NUM_JOBS=4 \
  -t mongodb-no-avx:8.0.17 .
```

**Build Arguments:**
- `MONGO_VERSION`: MongoDB version to build (default: `8.0.17`)
- `NUM_JOBS`: Number of parallel Bazel build jobs (leave empty for auto-detect)

**Note:** Building MongoDB from source takes a LONG time (2-6+ hours depending on your hardware) and requires significant resources:
- RAM: 16GB+ recommended (Bazel is memory-intensive)
- Disk: 50GB+ free space
- CPU: More cores = faster build

## Usage

### Standalone MongoDB

```bash
docker run -d \
  --name mongodb \
  -p 27017:27017 \
  -v mongodb-data:/data/db \
  mongodb-no-avx:8.0.17
```

### With Omada Controller (Bridge Networking)

Use `docker-compose.yml` for bridge networking mode:

```bash
docker compose up -d
```

This will:
1. Build the MongoDB image (if not already built)
2. Start MongoDB container
3. Start Omada Controller connected to the external MongoDB

### With Omada Controller (Host Networking - Recommended)

Use `docker-compose.host.yml` for host networking mode (better for device discovery):

```bash
docker compose -f docker-compose.host.yml up -d
```

## Configuration

### Environment Variables for Omada Controller

When using external MongoDB with Omada Controller, set these environment variables:

```yaml
environment:
  - MONGO_EXTERNAL=true
  - EAP_MONGOD_URI=mongodb://mongodb-host:27017/omada
```

### MongoDB Connection String

The connection string format is:
```
mongodb://[username:password@]host:port/database
```

For local/container networking:
- Bridge mode: `mongodb://omada-mongodb:27017/omada`
- Host mode: `mongodb://127.0.0.1:27017/omada`

## Available Versions

The build supports MongoDB 8.x versions. To build a specific version:

```bash
# MongoDB 8.0.17 LTS (default - recommended for Omada Controller 6.x)
docker build --build-arg MONGO_VERSION=8.0.17 -t mongodb-no-avx:8.0.17 .

# MongoDB 8.2.3 (latest)
docker build --build-arg MONGO_VERSION=8.2.3 -t mongodb-no-avx:8.2.3 .
```

**Note:** MongoDB 7.x and earlier use SCons build system. This Dockerfile is optimized for MongoDB 8.x which uses Bazel. For MongoDB 7.x, you may need to use the [original mongodb-without-avx project](https://github.com/alanedwardes/mongodb-without-avx).

## Volumes

- `/data/db` - MongoDB data directory
- `/data/configdb` - MongoDB configuration database (for sharded clusters)

## Ports

- `27017` - MongoDB default port

## Health Check

The image includes a basic health check. For external health checking:

```bash
docker exec mongodb mongod --eval "db.adminCommand('ping')"
```

## Troubleshooting

### Build Fails with Memory Errors

MongoDB/Bazel compilation is memory-intensive. If you run out of memory:
1. Increase Docker memory limits (16GB+ recommended)
2. Reduce `NUM_JOBS` build argument
3. Add swap space to your system
4. Use `--local_ram_resources` Bazel flag

### Build Fails with Disk Space Errors

Bazel builds require significant disk space:
1. Ensure 50GB+ free space
2. Clean Docker build cache: `docker builder prune`
3. Remove old images: `docker image prune -a`

### Container Won't Start

Check logs:
```bash
docker logs mongodb
```

Common issues:
- Data directory permissions
- Port already in use
- Insufficient system resources

### Omada Controller Can't Connect

1. Verify MongoDB is running: `docker ps`
2. Check MongoDB logs: `docker logs omada-mongodb`
3. Verify network connectivity between containers
4. Ensure `MONGO_EXTERNAL=true` is set
5. Verify the `EAP_MONGOD_URI` is correct

## How It Works

MongoDB 8.x builds with Bazel and uses architecture-specific compiler flags defined in:
```
bazel/toolchains/cc/mongo_linux/mongo_linux_cc_toolchain_config.bzl
```

The default for x86_64 is:
```
-march=sandybridge -mtune=generic -mprefer-vector-width=128
```

This build patches it to:
```
-march=x86-64-v2 -mtune=generic
```

This removes the AVX requirement while maintaining reasonable performance on modern CPUs.

## License

This project follows the same licensing as the original mongodb-without-avx project (GPL-3.0).

MongoDB itself is licensed under the Server Side Public License (SSPL).
