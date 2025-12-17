# Chapter 35: 가상화 및 클라우드 환경

## 난이도: ★★★★★ (고급)

## 학습 목표
- 가상화 환경에서 Minifilter 동작 이해
- Hyper-V 및 VMware 호환성 고려사항
- Windows 컨테이너 환경 지원
- Azure/AWS 클라우드 배포 전략

## 35.1 가상화 환경 개요

### 가상화 아키텍처와 드라이버

```
+------------------------------------------------------------------+
|                    가상화 환경 스택                                |
+------------------------------------------------------------------+
|                                                                   |
|  +----------------------------------------------------------+    |
|  |                    게스트 VM 1                            |    |
|  |  +------------------+     +------------------+            |    |
|  |  | 애플리케이션       |     | 애플리케이션       |            |    |
|  |  +------------------+     +------------------+            |    |
|  |           |                        |                      |    |
|  |  +----------------------------------------------+         |    |
|  |  |            Windows 커널                       |         |    |
|  |  |  +------------------+                         |         |    |
|  |  |  | Minifilter       | ←── 가상 디스크에서 동작 |         |    |
|  |  |  +------------------+                         |         |    |
|  |  +----------------------------------------------+         |    |
|  +----------------------------------------------------------+    |
|                          |                                        |
|  ========================|======================================  |
|                          | 가상화 계층                             |
|  ========================|======================================  |
|                          v                                        |
|  +----------------------------------------------------------+    |
|  |              하이퍼바이저 (Hyper-V, VMware)               |    |
|  +----------------------------------------------------------+    |
|                          |                                        |
|  +----------------------------------------------------------+    |
|  |              물리 하드웨어 (실제 디스크)                   |    |
|  +----------------------------------------------------------+    |
|                                                                   |
+------------------------------------------------------------------+
```

### 가상화가 드라이버에 미치는 영향

```c
// VirtualizationAwareness.c
// 가상화 환경 감지 및 적응
// Virtualization environment detection and adaptation

#include <fltKernel.h>
#include <ntddk.h>

// 가상화 환경 유형
// Virtualization environment type
typedef enum _VIRTUALIZATION_TYPE {
    VIRT_TYPE_PHYSICAL,      // 물리 환경
    VIRT_TYPE_HYPERV,        // Hyper-V
    VIRT_TYPE_VMWARE,        // VMware
    VIRT_TYPE_VIRTUALBOX,    // VirtualBox
    VIRT_TYPE_UNKNOWN_VM     // 알 수 없는 VM
} VIRTUALIZATION_TYPE;

// 가상화 환경 정보
// Virtualization environment info
typedef struct _VIRTUALIZATION_INFO {
    VIRTUALIZATION_TYPE Type;
    BOOLEAN IsVirtualMachine;
    BOOLEAN IsNestedVirtualization;
    BOOLEAN HasVirtualDisk;
    WCHAR HypervisorVendor[64];
} VIRTUALIZATION_INFO, *PVIRTUALIZATION_INFO;

// CPUID를 통한 하이퍼바이저 감지
// Detect hypervisor via CPUID
BOOLEAN
DetectHypervisor(
    _Out_ PVIRTUALIZATION_INFO VirtInfo
    )
{
    int cpuInfo[4] = { 0 };

    RtlZeroMemory(VirtInfo, sizeof(VIRTUALIZATION_INFO));

    // CPUID leaf 1: 하이퍼바이저 비트 확인
    // CPUID leaf 1: Check hypervisor bit
    __cpuid(cpuInfo, 1);

    // ECX bit 31 = 하이퍼바이저 존재
    // ECX bit 31 = Hypervisor present
    if (!(cpuInfo[2] & (1 << 31))) {
        VirtInfo->Type = VIRT_TYPE_PHYSICAL;
        VirtInfo->IsVirtualMachine = FALSE;
        return FALSE;
    }

    VirtInfo->IsVirtualMachine = TRUE;

    // CPUID leaf 0x40000000: 하이퍼바이저 벤더 확인
    // CPUID leaf 0x40000000: Check hypervisor vendor
    __cpuid(cpuInfo, 0x40000000);

    // 벤더 문자열 구성
    // Build vendor string
    char vendor[13] = { 0 };
    *(int*)(vendor + 0) = cpuInfo[1];
    *(int*)(vendor + 4) = cpuInfo[2];
    *(int*)(vendor + 8) = cpuInfo[3];

    // 벤더별 식별
    // Identify by vendor
    if (strncmp(vendor, "Microsoft Hv", 12) == 0) {
        VirtInfo->Type = VIRT_TYPE_HYPERV;
        RtlStringCbCopyW(VirtInfo->HypervisorVendor,
                         sizeof(VirtInfo->HypervisorVendor),
                         L"Microsoft Hyper-V");
    }
    else if (strncmp(vendor, "VMwareVMware", 12) == 0) {
        VirtInfo->Type = VIRT_TYPE_VMWARE;
        RtlStringCbCopyW(VirtInfo->HypervisorVendor,
                         sizeof(VirtInfo->HypervisorVendor),
                         L"VMware");
    }
    else if (strncmp(vendor, "VBoxVBoxVBox", 12) == 0) {
        VirtInfo->Type = VIRT_TYPE_VIRTUALBOX;
        RtlStringCbCopyW(VirtInfo->HypervisorVendor,
                         sizeof(VirtInfo->HypervisorVendor),
                         L"VirtualBox");
    }
    else {
        VirtInfo->Type = VIRT_TYPE_UNKNOWN_VM;
        // ASCII를 유니코드로 변환
        // Convert ASCII to Unicode
        for (int i = 0; i < 12 && vendor[i]; i++) {
            VirtInfo->HypervisorVendor[i] = (WCHAR)vendor[i];
        }
    }

    DbgPrint("[DrmFilter] 가상화 환경 감지: %ws\n", VirtInfo->HypervisorVendor);
    // [DrmFilter] Virtualization environment detected

    return TRUE;
}

// 가상 디스크 여부 확인
// Check if disk is virtual
BOOLEAN
IsVirtualDisk(
    _In_ PDEVICE_OBJECT DeviceObject
    )
{
    NTSTATUS status;
    STORAGE_PROPERTY_QUERY query;
    STORAGE_DEVICE_DESCRIPTOR descriptor;
    IO_STATUS_BLOCK ioStatus;
    PIRP irp;
    KEVENT event;

    RtlZeroMemory(&query, sizeof(query));
    query.PropertyId = StorageDeviceProperty;
    query.QueryType = PropertyStandardQuery;

    KeInitializeEvent(&event, NotificationEvent, FALSE);

    irp = IoBuildDeviceIoControlRequest(
        IOCTL_STORAGE_QUERY_PROPERTY,
        DeviceObject,
        &query,
        sizeof(query),
        &descriptor,
        sizeof(descriptor),
        FALSE,
        &event,
        &ioStatus
    );

    if (irp == NULL) {
        return FALSE;
    }

    status = IoCallDriver(DeviceObject, irp);

    if (status == STATUS_PENDING) {
        KeWaitForSingleObject(&event, Executive, KernelMode, FALSE, NULL);
        status = ioStatus.Status;
    }

    if (!NT_SUCCESS(status)) {
        return FALSE;
    }

    // 가상 디스크 벤더 확인
    // Check for virtual disk vendor
    // VBOX, VMware, Msft Virtual 등
    // VBOX, VMware, Msft Virtual, etc.

    return FALSE;  // 실제 구현에서는 벤더 문자열 검사
}

// 가상화 환경에 따른 설정 조정
// Adjust settings based on virtualization environment
VOID
AdjustSettingsForVirtualization(
    _In_ PVIRTUALIZATION_INFO VirtInfo
    )
{
    if (!VirtInfo->IsVirtualMachine) {
        return;
    }

    DbgPrint("[DrmFilter] 가상화 환경에 맞게 설정 조정\n");
    // [DrmFilter] Adjusting settings for virtualization

    switch (VirtInfo->Type) {
        case VIRT_TYPE_HYPERV:
            // Hyper-V 최적화
            // Hyper-V optimizations
            // - 더 큰 I/O 버퍼 사용
            // - Use larger I/O buffers
            g_Settings.IoBufferSize = 64 * 1024;
            break;

        case VIRT_TYPE_VMWARE:
            // VMware 최적화
            // VMware optimizations
            // - VMCI 통신 고려
            // - Consider VMCI communication
            g_Settings.IoBufferSize = 32 * 1024;
            break;

        default:
            // 기본 가상화 설정
            // Default virtualization settings
            g_Settings.IoBufferSize = 16 * 1024;
            break;
    }

    // 가상 환경에서는 타임아웃 증가
    // Increase timeouts in virtual environment
    g_Settings.DefaultTimeout.QuadPart *= 2;
}
```

## 35.2 Hyper-V 호환성

### Hyper-V 특수 고려사항

```c
// HyperVCompat.c
// Hyper-V 호환성 처리
// Hyper-V compatibility handling

#include <fltKernel.h>

// Hyper-V 세대 2 VM에서의 특수 처리
// Special handling for Hyper-V Generation 2 VMs

// VHDX 파일 필터링
// VHDX file filtering
BOOLEAN
IsVhdxFile(
    _In_ PCUNICODE_STRING FileName
    )
{
    static const UNICODE_STRING vhdxExt = RTL_CONSTANT_STRING(L".vhdx");
    static const UNICODE_STRING vhdExt = RTL_CONSTANT_STRING(L".vhd");
    static const UNICODE_STRING avhdxExt = RTL_CONSTANT_STRING(L".avhdx");

    UNICODE_STRING extension;

    // 확장자 추출
    // Extract extension
    USHORT lastDot = 0;
    for (USHORT i = 0; i < FileName->Length / sizeof(WCHAR); i++) {
        if (FileName->Buffer[i] == L'.') {
            lastDot = i;
        }
    }

    if (lastDot == 0) {
        return FALSE;
    }

    extension.Buffer = &FileName->Buffer[lastDot];
    extension.Length = FileName->Length - (lastDot * sizeof(WCHAR));
    extension.MaximumLength = extension.Length;

    return RtlEqualUnicodeString(&extension, &vhdxExt, TRUE) ||
           RtlEqualUnicodeString(&extension, &vhdExt, TRUE) ||
           RtlEqualUnicodeString(&extension, &avhdxExt, TRUE);
}

// Hyper-V 저장소 서비스 프로세스 확인
// Check if Hyper-V storage service process
BOOLEAN
IsHyperVStorageProcess(
    VOID
    )
{
    PEPROCESS process = PsGetCurrentProcess();
    PUNICODE_STRING processName = NULL;
    NTSTATUS status;
    BOOLEAN isHyperV = FALSE;

    status = SeLocateProcessImageName(process, &processName);

    if (NT_SUCCESS(status) && processName != NULL) {
        // vmmem, vmwp.exe 등 확인
        // Check for vmmem, vmwp.exe, etc.
        if (wcsstr(processName->Buffer, L"vmwp.exe") != NULL ||
            wcsstr(processName->Buffer, L"vmms.exe") != NULL ||
            wcsstr(processName->Buffer, L"vmmem") != NULL) {
            isHyperV = TRUE;
        }

        ExFreePool(processName);
    }

    return isHyperV;
}

// Hyper-V 환경에서의 PreCreate 처리
// PreCreate handling in Hyper-V environment
FLT_PREOP_CALLBACK_STATUS
HyperVAwarePreCreate(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID *CompletionContext
    )
{
    PFLT_FILE_NAME_INFORMATION nameInfo;
    NTSTATUS status;

    // Hyper-V 저장소 프로세스의 VHDX 접근 허용
    // Allow Hyper-V storage process access to VHDX
    if (IsHyperVStorageProcess()) {
        status = FltGetFileNameInformation(
            Data,
            FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,
            &nameInfo
        );

        if (NT_SUCCESS(status)) {
            FltParseFileNameInformation(nameInfo);

            if (IsVhdxFile(&nameInfo->Name)) {
                // Hyper-V VHDX 접근 - 필터링 건너뜀
                // Hyper-V VHDX access - skip filtering
                FltReleaseFileNameInformation(nameInfo);
                return FLT_PREOP_SUCCESS_NO_CALLBACK;
            }

            FltReleaseFileNameInformation(nameInfo);
        }
    }

    // 일반 처리 계속
    // Continue normal processing
    return FLT_PREOP_SUCCESS_WITH_CALLBACK;
}

// ReFS 파일 시스템 고려사항
// ReFS file system considerations
BOOLEAN
IsReFSVolume(
    _In_ PCFLT_RELATED_OBJECTS FltObjects
    )
{
    NTSTATUS status;
    PFILE_FS_ATTRIBUTE_INFORMATION fsAttrInfo;
    ULONG length;

    length = sizeof(FILE_FS_ATTRIBUTE_INFORMATION) + MAX_PATH * sizeof(WCHAR);
    fsAttrInfo = ExAllocatePool2(POOL_FLAG_PAGED, length, 'sFeR');

    if (fsAttrInfo == NULL) {
        return FALSE;
    }

    status = FltQueryVolumeInformation(
        FltObjects->Instance,
        NULL,
        fsAttrInfo,
        length,
        FileFsAttributeInformation
    );

    BOOLEAN isReFS = FALSE;

    if (NT_SUCCESS(status)) {
        // ReFS 확인
        // Check for ReFS
        if (wcsncmp(fsAttrInfo->FileSystemName, L"ReFS", 4) == 0) {
            isReFS = TRUE;
        }
    }

    ExFreePoolWithTag(fsAttrInfo, 'sFeR');

    return isReFS;
}
```

## 35.3 Windows 컨테이너

### 컨테이너 환경 감지

```c
// ContainerSupport.c
// Windows 컨테이너 지원
// Windows container support

#include <fltKernel.h>

// 컨테이너 유형
// Container type
typedef enum _CONTAINER_TYPE {
    CONTAINER_NONE,           // 컨테이너 아님
    CONTAINER_PROCESS,        // 프로세스 격리
    CONTAINER_HYPERV,         // Hyper-V 격리
    CONTAINER_SANDBOX         // Windows Sandbox
} CONTAINER_TYPE;

// 컨테이너 정보
// Container information
typedef struct _CONTAINER_INFO {
    CONTAINER_TYPE Type;
    GUID ContainerId;
    BOOLEAN IsIsolated;
    UNICODE_STRING SandboxPath;
} CONTAINER_INFO, *PCONTAINER_INFO;

// 현재 프로세스의 컨테이너 정보 가져오기
// Get container info for current process
NTSTATUS
GetContainerInfo(
    _Out_ PCONTAINER_INFO ContainerInfo
    )
{
    NTSTATUS status;
    PEPROCESS process;
    HANDLE processHandle;

    RtlZeroMemory(ContainerInfo, sizeof(CONTAINER_INFO));

    process = PsGetCurrentProcess();

    // 프로세스 토큰에서 컨테이너 SID 확인
    // Check container SID from process token
    PACCESS_TOKEN token = PsReferencePrimaryToken(process);

    if (token == NULL) {
        return STATUS_UNSUCCESSFUL;
    }

    // 토큰에서 컨테이너 ID 추출
    // Extract container ID from token
    // 실제 구현에서는 SeQueryInformationToken 사용
    // In real implementation, use SeQueryInformationToken

    PsDereferencePrimaryToken(token);

    // Job 오브젝트 확인
    // Check Job object
    PEJOB job = PsGetProcessJob(process);

    if (job != NULL) {
        // Silo 정보 확인
        // Check Silo info
        // Windows Server 컨테이너는 Server Silo 사용
        // Windows Server containers use Server Silo

        ContainerInfo->Type = CONTAINER_PROCESS;
        ContainerInfo->IsIsolated = TRUE;
    }

    return STATUS_SUCCESS;
}

// 컨테이너 내 파일 경로 변환
// Translate file path in container
NTSTATUS
TranslateContainerPath(
    _In_ PCUNICODE_STRING ContainerPath,
    _Out_ PUNICODE_STRING HostPath
    )
{
    // 컨테이너 경로를 호스트 경로로 변환
    // Translate container path to host path
    // 예: C:\ContainerSandbox\{GUID}\Files\... → 실제 호스트 경로
    // Example: C:\ContainerSandbox\{GUID}\Files\... → actual host path

    // 구현은 컨테이너 유형에 따라 다름
    // Implementation varies by container type

    return STATUS_NOT_IMPLEMENTED;
}

// 컨테이너 환경에서의 정책 적용
// Policy application in container environment
BOOLEAN
ShouldApplyPolicyInContainer(
    _In_ PCONTAINER_INFO ContainerInfo,
    _In_ PCUNICODE_STRING FilePath
    )
{
    // 컨테이너가 아니면 항상 적용
    // Always apply if not in container
    if (ContainerInfo->Type == CONTAINER_NONE) {
        return TRUE;
    }

    // Hyper-V 격리 컨테이너는 별도 VM이므로 독립 정책
    // Hyper-V isolated containers are separate VMs, independent policy
    if (ContainerInfo->Type == CONTAINER_HYPERV) {
        return TRUE;
    }

    // 프로세스 격리 컨테이너는 호스트와 파일 시스템 공유 가능
    // Process isolated containers can share filesystem with host
    // 정책은 호스트 경로 기준으로 적용
    // Apply policy based on host path

    return TRUE;
}
```

### Docker/Kubernetes 통합

```csharp
// ContainerOrchestration.cs
// 컨테이너 오케스트레이션 통합
// Container orchestration integration

using System.Text.Json;
using k8s;
using k8s.Models;

namespace DrmFilterService;

/// <summary>
/// Kubernetes 통합 서비스
/// Kubernetes integration service
/// </summary>
public sealed class KubernetesIntegration
{
    private readonly ILogger<KubernetesIntegration> _logger;
    private readonly Kubernetes _client;
    private readonly string _namespace;
    private readonly string _daemonSetName;

    public KubernetesIntegration(
        ILogger<KubernetesIntegration> logger,
        string kubeConfigPath,
        string @namespace = "drm-filter")
    {
        _logger = logger;
        _namespace = @namespace;
        _daemonSetName = "drm-filter-agent";

        var config = KubernetesClientConfiguration.BuildConfigFromConfigFile(kubeConfigPath);
        _client = new Kubernetes(config);
    }

    /// <summary>
    /// DaemonSet 배포
    /// Deploy DaemonSet
    /// </summary>
    public async Task DeployDaemonSetAsync()
    {
        var daemonSet = new V1DaemonSet
        {
            Metadata = new V1ObjectMeta
            {
                Name = _daemonSetName,
                NamespaceProperty = _namespace,
                Labels = new Dictionary<string, string>
                {
                    ["app"] = "drm-filter"
                }
            },
            Spec = new V1DaemonSetSpec
            {
                Selector = new V1LabelSelector
                {
                    MatchLabels = new Dictionary<string, string>
                    {
                        ["app"] = "drm-filter"
                    }
                },
                Template = new V1PodTemplateSpec
                {
                    Metadata = new V1ObjectMeta
                    {
                        Labels = new Dictionary<string, string>
                        {
                            ["app"] = "drm-filter"
                        }
                    },
                    Spec = new V1PodSpec
                    {
                        // Windows 노드에서만 실행
                        // Run only on Windows nodes
                        NodeSelector = new Dictionary<string, string>
                        {
                            ["kubernetes.io/os"] = "windows"
                        },
                        Containers = new List<V1Container>
                        {
                            new V1Container
                            {
                                Name = "drm-filter-agent",
                                Image = "myregistry/drm-filter-agent:latest",
                                SecurityContext = new V1SecurityContext
                                {
                                    // 권한 있는 컨테이너 필요 (드라이버 설치)
                                    // Privileged container required (driver install)
                                    WindowsOptions = new V1WindowsSecurityContextOptions
                                    {
                                        HostProcess = true,
                                        RunAsUserName = "NT AUTHORITY\\SYSTEM"
                                    }
                                },
                                VolumeMounts = new List<V1VolumeMount>
                                {
                                    new V1VolumeMount
                                    {
                                        Name = "driver-volume",
                                        MountPath = "C:\\Windows\\System32\\drivers"
                                    }
                                }
                            }
                        },
                        Volumes = new List<V1Volume>
                        {
                            new V1Volume
                            {
                                Name = "driver-volume",
                                HostPath = new V1HostPathVolumeSource
                                {
                                    Path = "C:\\Windows\\System32\\drivers",
                                    Type = "Directory"
                                }
                            }
                        }
                    }
                }
            }
        };

        try
        {
            await _client.CreateNamespacedDaemonSetAsync(daemonSet, _namespace);
            _logger.LogInformation("DaemonSet 배포됨: {Name}", _daemonSetName);
            // DaemonSet deployed
        }
        catch (k8s.Autorest.HttpOperationException ex) when (ex.Response.StatusCode == System.Net.HttpStatusCode.Conflict)
        {
            _logger.LogInformation("DaemonSet 이미 존재함, 업데이트 중...");
            // DaemonSet already exists, updating...
            await _client.ReplaceNamespacedDaemonSetAsync(daemonSet, _daemonSetName, _namespace);
        }
    }

    /// <summary>
    /// 노드별 상태 조회
    /// Get status by node
    /// </summary>
    public async Task<IEnumerable<NodeStatus>> GetNodeStatusesAsync()
    {
        var pods = await _client.ListNamespacedPodAsync(
            _namespace,
            labelSelector: "app=drm-filter");

        var statuses = new List<NodeStatus>();

        foreach (var pod in pods.Items)
        {
            statuses.Add(new NodeStatus
            {
                NodeName = pod.Spec.NodeName,
                PodName = pod.Metadata.Name,
                Phase = pod.Status.Phase,
                Ready = pod.Status.ContainerStatuses?.All(c => c.Ready) ?? false
            });
        }

        return statuses;
    }

    /// <summary>
    /// ConfigMap으로 정책 배포
    /// Deploy policies via ConfigMap
    /// </summary>
    public async Task DeployPoliciesAsync(IEnumerable<ProtectionPolicy> policies)
    {
        var configMap = new V1ConfigMap
        {
            Metadata = new V1ObjectMeta
            {
                Name = "drm-filter-policies",
                NamespaceProperty = _namespace
            },
            Data = new Dictionary<string, string>
            {
                ["policies.json"] = JsonSerializer.Serialize(policies)
            }
        };

        try
        {
            await _client.CreateNamespacedConfigMapAsync(configMap, _namespace);
        }
        catch (k8s.Autorest.HttpOperationException ex) when (ex.Response.StatusCode == System.Net.HttpStatusCode.Conflict)
        {
            await _client.ReplaceNamespacedConfigMapAsync(configMap, "drm-filter-policies", _namespace);
        }

        _logger.LogInformation("정책 ConfigMap 배포됨");
        // Policies ConfigMap deployed
    }
}

public sealed class NodeStatus
{
    public required string NodeName { get; init; }
    public required string PodName { get; init; }
    public string? Phase { get; init; }
    public bool Ready { get; init; }
}
```

## 35.4 클라우드 환경 배포

### Azure VM 배포

```csharp
// AzureDeployment.cs
// Azure VM 드라이버 배포
// Azure VM driver deployment

using Azure.Identity;
using Azure.ResourceManager;
using Azure.ResourceManager.Compute;
using Azure.ResourceManager.Compute.Models;

namespace DrmFilterService;

/// <summary>
/// Azure VM 배포 관리자
/// Azure VM deployment manager
/// </summary>
public sealed class AzureDeploymentManager
{
    private readonly ILogger<AzureDeploymentManager> _logger;
    private readonly ArmClient _armClient;
    private readonly string _subscriptionId;

    public AzureDeploymentManager(
        ILogger<AzureDeploymentManager> logger,
        string subscriptionId)
    {
        _logger = logger;
        _subscriptionId = subscriptionId;

        // DefaultAzureCredential 사용 (Managed Identity, CLI 등)
        // Using DefaultAzureCredential (Managed Identity, CLI, etc.)
        _armClient = new ArmClient(new DefaultAzureCredential());
    }

    /// <summary>
    /// VM에 드라이버 설치 (Custom Script Extension)
    /// Install driver on VM (Custom Script Extension)
    /// </summary>
    public async Task InstallDriverOnVmAsync(
        string resourceGroup,
        string vmName,
        string driverBlobUrl,
        string sasToken)
    {
        _logger.LogInformation("VM에 드라이버 설치 중: {VmName}", vmName);
        // Installing driver on VM

        var subscription = _armClient.GetSubscriptionResource(
            new Azure.Core.ResourceIdentifier($"/subscriptions/{_subscriptionId}"));

        var resourceGroupResource = await subscription
            .GetResourceGroupAsync(resourceGroup);

        var vm = await resourceGroupResource.Value
            .GetVirtualMachineAsync(vmName);

        // Custom Script Extension으로 드라이버 설치
        // Install driver via Custom Script Extension
        var extensionData = new VirtualMachineExtensionData(vm.Value.Data.Location)
        {
            Publisher = "Microsoft.Compute",
            ExtensionType = "CustomScriptExtension",
            TypeHandlerVersion = "1.10",
            AutoUpgradeMinorVersion = true,
            Settings = BinaryData.FromObjectAsJson(new
            {
                fileUris = new[] { $"{driverBlobUrl}?{sasToken}" },
                commandToExecute = @"powershell -ExecutionPolicy Bypass -File Install-DrmFilter.ps1"
            })
        };

        await vm.Value.GetVirtualMachineExtensions()
            .CreateOrUpdateAsync(
                Azure.WaitUntil.Completed,
                "DrmFilterInstall",
                extensionData);

        _logger.LogInformation("드라이버 설치 완료: {VmName}", vmName);
        // Driver installation complete
    }

    /// <summary>
    /// VM Scale Set에 드라이버 배포
    /// Deploy driver to VM Scale Set
    /// </summary>
    public async Task DeployToScaleSetAsync(
        string resourceGroup,
        string scaleSetName,
        string driverBlobUrl)
    {
        _logger.LogInformation("Scale Set에 드라이버 배포: {ScaleSet}", scaleSetName);
        // Deploying driver to Scale Set

        var subscription = _armClient.GetSubscriptionResource(
            new Azure.Core.ResourceIdentifier($"/subscriptions/{_subscriptionId}"));

        var resourceGroupResource = await subscription
            .GetResourceGroupAsync(resourceGroup);

        var vmss = await resourceGroupResource.Value
            .GetVirtualMachineScaleSets()
            .GetAsync(scaleSetName);

        // 확장 추가/업데이트
        // Add/update extension
        var extensionProfile = vmss.Value.Data.VirtualMachineProfile.ExtensionProfile;

        extensionProfile ??= new VirtualMachineScaleSetExtensionProfile();
        extensionProfile.Extensions ??= new List<VirtualMachineScaleSetExtensionData>();

        var drmExtension = new VirtualMachineScaleSetExtensionData
        {
            Name = "DrmFilterExtension",
            Publisher = "Microsoft.Compute",
            ExtensionType = "CustomScriptExtension",
            TypeHandlerVersion = "1.10",
            AutoUpgradeMinorVersion = true,
            Settings = BinaryData.FromObjectAsJson(new
            {
                fileUris = new[] { driverBlobUrl },
                commandToExecute = @"powershell -ExecutionPolicy Bypass -File Install-DrmFilter.ps1"
            })
        };

        // 기존 확장 교체 또는 추가
        // Replace or add existing extension
        var existing = extensionProfile.Extensions
            .FirstOrDefault(e => e.Name == "DrmFilterExtension");

        if (existing != null)
        {
            extensionProfile.Extensions.Remove(existing);
        }

        extensionProfile.Extensions.Add(drmExtension);

        // VMSS 업데이트
        // Update VMSS
        var updateData = new VirtualMachineScaleSetPatch
        {
            VirtualMachineProfile = vmss.Value.Data.VirtualMachineProfile
        };

        await vmss.Value.UpdateAsync(Azure.WaitUntil.Completed, updateData);

        // 인스턴스 업그레이드 (Rolling update)
        // Upgrade instances (Rolling update)
        var instances = vmss.Value.GetVirtualMachineScaleSetVms();
        var instanceIds = new List<string>();

        await foreach (var instance in instances.GetAllAsync())
        {
            instanceIds.Add(instance.Data.InstanceId);
        }

        // 배치로 업그레이드
        // Upgrade in batches
        const int batchSize = 5;
        for (int i = 0; i < instanceIds.Count; i += batchSize)
        {
            var batch = instanceIds.Skip(i).Take(batchSize).ToArray();
            await vmss.Value.UpdateInstancesAsync(
                Azure.WaitUntil.Completed,
                new VirtualMachineScaleSetVmInstanceRequiredIds(batch));

            _logger.LogInformation("배치 업그레이드 완료: {Start}-{End}",
                i, Math.Min(i + batchSize, instanceIds.Count));
            // Batch upgrade complete
        }
    }
}
```

### AWS EC2 배포

```csharp
// AwsDeployment.cs
// AWS EC2 드라이버 배포
// AWS EC2 driver deployment

using Amazon.EC2;
using Amazon.EC2.Model;
using Amazon.SimpleSystemsManagement;
using Amazon.SimpleSystemsManagement.Model;

namespace DrmFilterService;

/// <summary>
/// AWS EC2 배포 관리자
/// AWS EC2 deployment manager
/// </summary>
public sealed class AwsDeploymentManager
{
    private readonly ILogger<AwsDeploymentManager> _logger;
    private readonly IAmazonEC2 _ec2Client;
    private readonly IAmazonSimpleSystemsManagement _ssmClient;

    public AwsDeploymentManager(
        ILogger<AwsDeploymentManager> logger,
        IAmazonEC2 ec2Client,
        IAmazonSimpleSystemsManagement ssmClient)
    {
        _logger = logger;
        _ec2Client = ec2Client;
        _ssmClient = ssmClient;
    }

    /// <summary>
    /// SSM Run Command로 드라이버 설치
    /// Install driver via SSM Run Command
    /// </summary>
    public async Task InstallDriverViaSsmAsync(
        IEnumerable<string> instanceIds,
        string s3BucketUrl)
    {
        _logger.LogInformation("SSM으로 드라이버 설치 시작");
        // Starting driver installation via SSM

        var installScript = $@"
            # 드라이버 패키지 다운로드
            # Download driver package
            $downloadPath = 'C:\Temp\DrmFilter'
            New-Item -ItemType Directory -Force -Path $downloadPath

            # S3에서 다운로드
            # Download from S3
            Read-S3Object -BucketName {s3BucketUrl} -Key 'DrmFilter/DrmFilterPackage.zip' -File '$downloadPath\package.zip'

            # 압축 해제
            # Extract
            Expand-Archive -Path '$downloadPath\package.zip' -DestinationPath $downloadPath -Force

            # 설치 스크립트 실행
            # Run install script
            & '$downloadPath\Install-DrmFilter.ps1'

            # 정리
            # Cleanup
            Remove-Item -Path $downloadPath -Recurse -Force
        ";

        var command = new SendCommandRequest
        {
            DocumentName = "AWS-RunPowerShellScript",
            InstanceIds = instanceIds.ToList(),
            Parameters = new Dictionary<string, List<string>>
            {
                ["commands"] = new List<string> { installScript }
            },
            TimeoutSeconds = 600,
            Comment = "DRM Filter Driver Installation"
        };

        var response = await _ssmClient.SendCommandAsync(command);

        _logger.LogInformation("SSM 명령 전송됨: {CommandId}", response.Command.CommandId);
        // SSM command sent

        // 완료 대기
        // Wait for completion
        await WaitForCommandCompletionAsync(response.Command.CommandId, instanceIds);
    }

    private async Task WaitForCommandCompletionAsync(
        string commandId,
        IEnumerable<string> instanceIds)
    {
        var allCompleted = false;
        var maxWaitTime = TimeSpan.FromMinutes(10);
        var startTime = DateTime.UtcNow;

        while (!allCompleted && (DateTime.UtcNow - startTime) < maxWaitTime)
        {
            await Task.Delay(5000);

            allCompleted = true;

            foreach (var instanceId in instanceIds)
            {
                var invocation = await _ssmClient.GetCommandInvocationAsync(
                    new GetCommandInvocationRequest
                    {
                        CommandId = commandId,
                        InstanceId = instanceId
                    });

                if (invocation.Status == CommandInvocationStatus.InProgress ||
                    invocation.Status == CommandInvocationStatus.Pending)
                {
                    allCompleted = false;
                    break;
                }

                if (invocation.Status == CommandInvocationStatus.Failed)
                {
                    _logger.LogError(
                        "인스턴스 {Instance} 설치 실패: {Error}",
                        instanceId,
                        invocation.StandardErrorContent);
                    // Instance installation failed
                }
            }
        }

        if (!allCompleted)
        {
            _logger.LogWarning("일부 인스턴스 설치 타임아웃");
            // Some instance installations timed out
        }
    }

    /// <summary>
    /// Auto Scaling Group에 배포
    /// Deploy to Auto Scaling Group
    /// </summary>
    public async Task DeployToAutoScalingGroupAsync(
        string autoScalingGroupName,
        string amiWithDriver)
    {
        _logger.LogInformation("ASG에 드라이버 AMI 배포: {ASG}", autoScalingGroupName);
        // Deploying driver AMI to ASG

        // Launch Template 업데이트 또는 새 AMI로 인스턴스 교체
        // Update Launch Template or replace instances with new AMI

        // 실제 구현에서는 AutoScaling 클라이언트 사용
        // In real implementation, use AutoScaling client

        await Task.CompletedTask;
    }
}
```

## 35.5 클라우드 스토리지 통합

### Azure Files/AWS EFS 지원

```c
// CloudStorageFilter.c
// 클라우드 스토리지 파일 시스템 필터링
// Cloud storage file system filtering

#include <fltKernel.h>

// 클라우드 스토리지 유형
// Cloud storage type
typedef enum _CLOUD_STORAGE_TYPE {
    CLOUD_NONE,
    CLOUD_AZURE_FILES,     // Azure Files (SMB)
    CLOUD_AZURE_BLOB,      // Azure Blob (NFS/REST)
    CLOUD_AWS_EFS,         // AWS EFS
    CLOUD_AWS_FSX,         // AWS FSx
    CLOUD_ONEDRIVE         // OneDrive (클라우드 파일)
} CLOUD_STORAGE_TYPE;

// 클라우드 스토리지 감지
// Detect cloud storage
CLOUD_STORAGE_TYPE
DetectCloudStorage(
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_ PCUNICODE_STRING VolumeName
    )
{
    // Azure Files: \\*.file.core.windows.net\*
    static const UNICODE_STRING azureFilesPattern =
        RTL_CONSTANT_STRING(L"\\\\*.file.core.windows.net\\");

    // AWS EFS: \\fs-*.efs.*.amazonaws.com\*
    // (Windows용 EFS 마운트)

    // OneDrive: OneDrive 동기화 폴더
    // OneDrive: OneDrive sync folder

    if (VolumeName != NULL) {
        // Azure Files 확인
        // Check for Azure Files
        if (wcsstr(VolumeName->Buffer, L".file.core.windows.net") != NULL) {
            return CLOUD_AZURE_FILES;
        }

        // AWS FSx 확인
        // Check for AWS FSx
        if (wcsstr(VolumeName->Buffer, L".fsx.") != NULL &&
            wcsstr(VolumeName->Buffer, L".amazonaws.com") != NULL) {
            return CLOUD_AWS_FSX;
        }
    }

    return CLOUD_NONE;
}

// 클라우드 파일 속성 확인 (OneDrive 등)
// Check cloud file attributes (OneDrive, etc.)
BOOLEAN
IsCloudFile(
    _In_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects
    )
{
    NTSTATUS status;
    FILE_ATTRIBUTE_TAG_INFORMATION tagInfo;

    // FILE_ATTRIBUTE_RECALL_ON_DATA_ACCESS 확인
    // Check FILE_ATTRIBUTE_RECALL_ON_DATA_ACCESS
    status = FltQueryInformationFile(
        FltObjects->Instance,
        FltObjects->FileObject,
        &tagInfo,
        sizeof(tagInfo),
        FileAttributeTagInformation,
        NULL
    );

    if (NT_SUCCESS(status)) {
        // 클라우드 파일 (데이터가 클라우드에 있음)
        // Cloud file (data is in cloud)
        if (tagInfo.FileAttributes & FILE_ATTRIBUTE_RECALL_ON_DATA_ACCESS) {
            return TRUE;
        }

        // 오프라인 파일
        // Offline file
        if (tagInfo.FileAttributes & FILE_ATTRIBUTE_OFFLINE) {
            return TRUE;
        }
    }

    return FALSE;
}

// 클라우드 스토리지에서의 정책 적용
// Policy application on cloud storage
FLT_PREOP_CALLBACK_STATUS
CloudAwarePreCreate(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID *CompletionContext
    )
{
    CLOUD_STORAGE_TYPE cloudType;
    UNICODE_STRING volumeName = { 0 };

    // 볼륨 이름 가져오기
    // Get volume name
    FltGetVolumeName(FltObjects->Volume, NULL, &volumeName.MaximumLength);

    if (volumeName.MaximumLength > 0) {
        volumeName.Buffer = ExAllocatePool2(
            POOL_FLAG_PAGED,
            volumeName.MaximumLength,
            'loVc'
        );

        if (volumeName.Buffer != NULL) {
            FltGetVolumeName(FltObjects->Volume, &volumeName, NULL);
        }
    }

    cloudType = DetectCloudStorage(FltObjects, &volumeName);

    if (volumeName.Buffer != NULL) {
        ExFreePoolWithTag(volumeName.Buffer, 'loVc');
    }

    if (cloudType != CLOUD_NONE) {
        DbgPrint("[DrmFilter] 클라우드 스토리지 접근: Type=%d\n", cloudType);
        // [DrmFilter] Cloud storage access

        // 클라우드 스토리지별 특수 처리
        // Special handling by cloud storage type
        switch (cloudType) {
            case CLOUD_AZURE_FILES:
                // Azure Files: SMB 프로토콜 고려
                // Azure Files: Consider SMB protocol
                break;

            case CLOUD_ONEDRIVE:
                // OneDrive: 동기화 상태 고려
                // OneDrive: Consider sync state
                break;

            default:
                break;
        }
    }

    return FLT_PREOP_SUCCESS_WITH_CALLBACK;
}
```

## 35.6 정리

이 챕터에서 학습한 내용:

1. **가상화 환경**
   - 하이퍼바이저 감지
   - 가상 디스크 식별
   - 성능 최적화 조정

2. **Hyper-V 호환성**
   - VHDX 파일 처리
   - Hyper-V 서비스 프로세스 식별
   - ReFS 파일 시스템 지원

3. **Windows 컨테이너**
   - 컨테이너 환경 감지
   - Docker/Kubernetes 통합
   - DaemonSet 배포

4. **클라우드 배포**
   - Azure VM/VMSS 배포
   - AWS EC2/SSM 통합
   - Auto Scaling 지원

5. **클라우드 스토리지**
   - Azure Files 감지
   - OneDrive 클라우드 파일
   - 오프라인 파일 처리

다음 챕터에서는 **보안 강화 및 고급 기법**을 다룹니다.

---

**다음 챕터 예고**: Chapter 36에서는 안티 탬퍼링, 자가 보호, 위협 탐지 등 보안 강화 기법을 학습합니다.
