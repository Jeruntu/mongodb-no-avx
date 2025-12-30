# MongoDB without AVX (mongodb-no-avx)

Docker image to build MongoDB from source without AVX CPU instruction requirements. This is designed for use with [mbentley/docker-omada-controller](https://github.com/mbentley/docker-omada-controller) on older CPUs that don't support AVX instructions.

Based on the work from [alanedwardes/mongodb-without-avx](https://github.com/alanedwardes/mongodb-without-avx), updated for MongoDB 7.x with SCons build system.

## Pre-built Images

Pre-built images are available from GitHub Container Registry:

```bash
docker pull ghcr.io/fenio/mongodb-no-avx:7.0.16
docker pull ghcr.io/fenio/mongodb-no-avx:7.0
docker pull ghcr.io/fenio/mongodb-no-avx:7
docker pull ghcr.io/fenio/mongodb-no-avx:latest  # Same as 7.0.16
```

## Why?

MongoDB 5.0+ requires AVX (Advanced Vector Extensions) CPU instructions by default. Many older CPUs (pre-2011) and some virtualization platforms don't support AVX. This project builds MongoDB from source with AVX optimizations disabled.

### Technical Details

MongoDB 7.x uses SCons build system with the `experimental-optimization` option that defaults to `+sandybridge`, which sets `-march=sandybridge` compiler flag requiring AVX support. This build patches the SConstruct to remove the sandybridge default, allowing MongoDB to use a generic x86-64 baseline:

- Does NOT require AVX instructions
- Works on Intel Westmere (2010) and newer, AMD Bulldozer (2011) and newer
- Compatible with virtualization platforms that don't expose AVX

## Building the Image

### Quick Build

```bash
docker build -t mongodb-no-avx:7.0.16 .
```

### Build with Specific Version

```bash
docker build \
  --build-arg MONGO_VERSION=7.0.16 \
  --build-arg NUM_JOBS=4 \
  -t mongodb-no-avx:7.0.16 .
```

**Build Arguments:**
- `MONGO_VERSION`: MongoDB version to build (default: `7.0.16`)
- `NUM_JOBS`: Number of parallel SCons build jobs (leave empty for auto-detect)

**Note:** Building MongoDB from source takes a LONG time (2-6+ hours depending on your hardware) and requires significant resources:
- RAM: 8GB+ recommended
- Disk: 30GB+ free space
- CPU: More cores = faster build

## Usage

### Standalone MongoDB

```bash
docker run -d \
  --name mongodb \
  -p 27017:27017 \
  -v mongodb-data:/data/db \
  ghcr.io/fenio/mongodb-no-avx:7.0.16
```

### With Omada Controller (Bridge Networking)

Use `docker-compose.yml` for bridge networking mode:

```bash
docker compose up -d
```

This will:
1. Pull/build the MongoDB image
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

The build supports MongoDB 7.0.x versions. To build a specific version:

```bash
# MongoDB 7.0.16 (default - recommended for Omada Controller)
docker build --build-arg MONGO_VERSION=7.0.16 -t mongodb-no-avx:7.0.16 .
```

**Note:** MongoDB 8.x uses Bazel build system which has enterprise module dependencies that complicate community builds. This Dockerfile is optimized for MongoDB 7.x which uses the SCons build system.

## Volumes

- `/data/db` - MongoDB data directory
- `/data/configdb` - MongoDB configuration database (for sharded clusters)

## Ports

- `27017` - MongoDB default port

## Troubleshooting

### Build Fails with Memory Errors

MongoDB/SCons compilation can be memory-intensive. If you run out of memory:
1. Increase Docker memory limits (8GB+ recommended)
2. Reduce `NUM_JOBS` build argument
3. Add swap space to your system

### Build Fails with Disk Space Errors

SCons builds require significant disk space:
1. Ensure 30GB+ free space
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

MongoDB 7.x builds with SCons and uses the `experimental-optimization` option in `SConstruct` that defaults to `['+sandybridge']`. This sets architecture-specific compiler flags:

```
-march=sandybridge -mtune=generic -mprefer-vector-width=128
```

The patch changes the default to `[]` (empty), which allows MongoDB to use the standard x86-64 baseline without AVX requirements.

## License

This project follows the same licensing as the original mongodb-without-avx project (GPL-3.0).

MongoDB itself is licensed under the Server Side Public License (SSPL).
