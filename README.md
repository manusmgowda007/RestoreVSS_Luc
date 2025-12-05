# RestoreVSS - Fast VSS Snapshot Restore Utility

A high-performance C++ command-line utility for restoring data from VSS (Volume Shadow Copy Service) snapshots to empty disk partitions.

## Features

- **Fast Blocked I/O**: Optimized for maximum throughput with configurable block sizes
- **Overlapped I/O**: Asynchronous I/O operations with configurable queue depth for parallel read/write operations
- **Sector Alignment**: Automatic detection and alignment to device sector boundaries
- **Progress Monitoring**: Real-time progress updates with throughput statistics
- **Error Handling**: Comprehensive error checking and reporting

## Prerequisites

- Windows 10 or later
- Visual Studio 2019 or later (for building)
- Administrator privileges (required for raw disk access)

## Building

Open `RestoreVSS.sln` in Visual Studio and build the solution, or use the command line:

```cmd
cl /EHsc /std:c++17 /O2 RestoreVSS\RestoreVSS.cpp /Fe:RestoreVSS.exe
```

## Usage

### Creating a VSS Snapshot

Before using this utility, create a VSS snapshot using PowerShell (as Administrator):

```powershell
powershell.exe -Command (gwmi -list win32_shadowcopy).Create('E:\','ClientAccessable')
```

### Finding the Shadow Volume Path

After creating a snapshot, find the shadow volume path. You can list shadow copies:

```powershell
Get-WmiObject Win32_ShadowCopy | Select-Object DeviceObject, VolumeName, InstallDate
```

The `DeviceObject` field contains the path you need (e.g., `\\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1`).

### Preparing the Target Partition

Ensure you have a clean disk with an empty partition matching the snapshot size. You can use `diskpart`:

```cmd
diskpart
list disk
select disk <disk_number>
list partition
select partition <partition_number>
format fs=ntfs quick
```

### Running RestoreVSS

```cmd
RestoreVSS.exe --shadow <shadow_path> --target <target_path> [options]
```

#### Parameters

- `--shadow <path>` (Required): Path to the shadow volume
  - Example: `\\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1`
- `--target <path>` (Required): Path to the target partition
  - Example: `\\.\PhysicalDrive1` or `\\.\E:`
- `--block <size>` (Optional): Block size for I/O operations
  - Default: `8M`
  - Suffixes: `K` (kilobytes), `M` (megabytes), `G` (gigabytes)
  - Example: `--block 16M`
- `--qdepth <n>` (Optional): I/O queue depth for overlapped I/O
  - Default: `4`
  - Range: 1-32
  - Higher values can improve performance on fast SSDs
  - Example: `--qdepth 8`
- `--no-overlapped` (Optional): Disable overlapped I/O (use sequential mode)
  - Use this if you encounter issues with overlapped I/O
- `--verbose` (Optional): Enable verbose output with detailed information

#### Example Commands

```cmd
# Basic usage with defaults
RestoreVSS.exe --shadow \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1 --target \\.\PhysicalDrive1

# High-performance configuration
RestoreVSS.exe --shadow \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1 --target \\.\PhysicalDrive1 --block 16M --qdepth 8

# With verbose output
RestoreVSS.exe --shadow \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1 --target \\.\PhysicalDrive1 --block 16M --qdepth 8 --verbose
```

## Performance Tuning

### Block Size
- **Smaller blocks (1-4MB)**: Better for slower storage, lower memory usage
- **Larger blocks (8-32MB)**: Better for fast SSDs, higher throughput
- **Optimal**: Usually 8-16MB for most modern storage

### Queue Depth
- **Lower (1-4)**: Better for slower storage or HDDs
- **Higher (8-16)**: Better for fast SSDs with high IOPS
- **Optimal**: Start with 4, increase if CPU and storage can handle it

### Overlapped I/O
- Enabled by default for maximum performance
- Disable with `--no-overlapped` if you encounter compatibility issues
- Sequential mode is simpler but typically slower

## Output

The utility provides:
- Real-time progress updates (updated every 64MB)
- Final statistics including:
  - Total bytes copied
  - Number of read/write operations
  - Elapsed time
  - Average throughput (MB/s)

Example output:
```
Progress:  45.23% | Copied:   47.32 GB | Speed:  523.45 MB/s | Time:  92.3 s
========================================
Copy completed successfully!
========================================
Bytes copied:  104.86 GB
Read operations:  13422
Write operations: 13422
Elapsed time:  204.32 seconds
Throughput:     512.89 MB/s
========================================
```

## Important Notes

1. **Administrator Rights**: This utility requires administrator privileges to access raw disk devices
2. **Data Loss Warning**: Writing to a target partition will overwrite all existing data
3. **Partition Size**: Ensure the target partition is at least as large as the source volume
4. **Snapshot Persistence**: VSS snapshots are temporary and may be deleted by the system
5. **File System**: The utility performs raw block copying and does not preserve file system metadata in the same way as file-level copying

## Troubleshooting

### "Failed to open shadow volume"
- Verify the snapshot exists and the path is correct
- Ensure you're running as Administrator
- Check that the snapshot hasn't been deleted

### "Failed to open target partition"
- Verify the target path is correct
- Ensure you have write access to the target
- Check that the partition exists and is accessible

### "Target partition is smaller than source volume"
- The target partition must be at least as large as the source
- Use `diskpart` to check partition sizes
- Create a larger partition if needed

### Low Performance
- Try increasing block size (`--block 16M` or `--block 32M`)
- Try increasing queue depth (`--qdepth 8` or `--qdepth 16`)
- Ensure both source and target are on fast storage (SSDs)
- Check for other processes accessing the disks


