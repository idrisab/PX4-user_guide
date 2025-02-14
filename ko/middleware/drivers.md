# 드라이버 개발

PX4 장치 드라이버는 [장치](https://github.com/PX4/PX4-Autopilot/tree/master/src/lib/drivers/device) 프레임워크를 기반으로 합니다.

## 드라이버 생성

PX4는 [uORB](../middleware/uorb.md)의 데이터를 거의 독점적으로 사용합니다. 일반적인 주변 장치 유형에 대한 드라이버는 올바른 uORB 메시지(예: 자이로, 가속도계, 압력 센서 등)를 게시하여야 합니다.

새 드라이버를 만드는 가장 좋은 방법은 템플릿과 유사한 드라이버로 시작하는 것입니다([src/drivers](https://github.com/PX4/PX4-Autopilot/tree/master/src/drivers) 참조).

:::note
특정 I/O 버스 및 센서 작업에 대한 자세한 정보는 [센서 및 액추에이터 버스](../sensor_bus/README.md) 섹션을 참고하십시오.
:::

:::note
올바른 uORB 주제를 게시하는 것은 드라이버가 *준수해야* 하는 유일한 패턴입니다.
:::

## 핵심 아키텍처

PX4는 [반응형 시스템](../concept/architecture.md)이며, [uORB](../middleware/uorb.md) 게시/구독을 사용하여 메시지를 전송합니다. 파일 핸들은 시스템의 핵심 작업에 필요하지 않거나 사용되지 않습니다. 두 가지 주요 API가 사용됩니다.

* PX4가 실행되는 시스템에 따라 파일, 네트워크 또는 공유 메모리 백엔드가 있는 게시/구독 시스템.
* 장치를 열거하고 구성을 가져오거나 설정할 수 있는 전역 장치 레지스트리. 이것은 연결 목록이나 파일 시스템에 대한 매핑처럼 간단할 수 있습니다.

## 장치 ID

PX4는 장치 ID를 사용하여 시스템 전체에서 개별 센서를 일관되게 식별합니다. 이러한 ID는 구성 매개변수에 저장되며, 센서 보정 값을 일치시키고 어떤 센서가 어떤 로그 파일 항목에 기록되는 지 결정합니다.

센서의 순서(예: `/dev/mag0` 및 대체 `/dev/mag1`가 있는 경우)는 우선순위를 결정하지 않습니다. 높은 우선순위는 대신 게시된 uORB 주제입니다.

### 디코딩 예제

시스템에 3개의 자력계가 있는 경우에는 비행 로그(.px4log)를 사용하여 매개변수를 덤프합니다. 세 개의 매개변수는 센서 ID를 인코딩하고, `MAG_PRIME`은 어떤 자력계가 기본 센서로 선택되었는 지 식별합니다. 각 MAGx_ID는 24비트 숫자이며, 수동 디코딩을 하가 위하여 왼쪽에 0을 채워야 합니다.


```
CAL_MAG0_ID = 73225.0
CAL_MAG1_ID = 66826.0
CAL_MAG2_ID = 263178.0
CAL_MAG_PRIME = 73225.0
```

이것은 주소 `0x1E`의 버스 1, I2C를 통해 연결된 외부 HMC5983입니다. 로그 파일에 `IMU.MagX`로 표시됩니다.

```
# device ID 73225 in 24-bit binary:
00000001  00011110  00001 001

# decodes to:
HMC5883   0x1E    bus 1 I2C
```

이것은 SPI, 버스 1, 슬레이브 선택 슬롯 5를 통하여 연결된 내부 HMC5983입니다. 로그 파일에 `IMU1.MagX`로 표시됩니다.

```
# device ID 66826 in 24-bit binary:
00000001  00000101  00001 010

# decodes to:
HMC5883   dev 5   bus 1 SPI
```

그리고 이것은 SPI, 버스 1, 슬레이브 선택 슬롯 4를 통하여 연결된 내부 MPU9250 자력계입니다. 로그 파일에 `IMU2.MagX`로 표시됩니다.

```
# device ID 263178 in 24-bit binary:
00000100  00000100  00001 010

#decodes to:
MPU9250   dev 4   bus 1 SPI
```

### 장치 ID 인코딩

장치 ID는 이 형식에 따른 24비트 숫자입니다. 첫 번째 필드는 위의 디코딩 예에서 최하위 비트입니다.

```C
struct DeviceStructure {
  enum DeviceBusType bus_type : 3;
  uint8_t bus: 5;    // which instance of the bus type
  uint8_t address;   // address on the bus (eg. I2C address)
  uint8_t devtype;   // device class specific device type
};
```
`bus_type`은 다음과 같이 디코딩됩니다.

```C
enum DeviceBusType {
  DeviceBusType_UNKNOWN = 0,
  DeviceBusType_I2C     = 1,
  DeviceBusType_SPI     = 2,
  DeviceBusType_UAVCAN  = 3,
};
```

`devtype`은 다음과 같이 디코딩됩니다.

```C
#define DRV_MAG_DEVTYPE_HMC5883  0x01
#define DRV_MAG_DEVTYPE_LSM303D  0x02
#define DRV_MAG_DEVTYPE_ACCELSIM 0x03
#define DRV_MAG_DEVTYPE_MPU9250  0x04
#define DRV_ACC_DEVTYPE_LSM303D  0x11
#define DRV_ACC_DEVTYPE_BMA180   0x12
#define DRV_ACC_DEVTYPE_MPU6000  0x13
#define DRV_ACC_DEVTYPE_ACCELSIM 0x14
#define DRV_ACC_DEVTYPE_GYROSIM  0x15
#define DRV_ACC_DEVTYPE_MPU9250  0x16
#define DRV_GYR_DEVTYPE_MPU6000  0x21
#define DRV_GYR_DEVTYPE_L3GD20   0x22
#define DRV_GYR_DEVTYPE_GYROSIM  0x23
#define DRV_GYR_DEVTYPE_MPU9250  0x24
#define DRV_RNG_DEVTYPE_MB12XX   0x31
#define DRV_RNG_DEVTYPE_LL40LS   0x32
```

## 디버깅

일반적인 디버깅 주제는 [디버깅/로깅](../debug/README.md)을 참고하십시오.

### 상세 로깅

드라이버(및 기타 모듈)는 기본적으로 최소한의 자세한 로그 문자열을 출력합니다(예: `PX4_DEBUG`, `PX4_WARN`, `PX4_ERR` 등).

로그 상세도는 빌드 시 `RELEASE_BUILD`(기본값), `DEBUG_BUILD`(상세) 또는 `TRACE_BUILD`(매우 상세) 매크로를 사용하여 정의됩니다.

드라이버 `px4_add_module` 함수(**CMakeLists.txt**)에서 `COMPILE_FLAGS`를 사용하여 로깅 수준을 변경합니다. 아래 코드 조각은 단일 모듈 또는 드라이버에 대해 DEBUG_BUILD 수준 디버깅을 활성화하에 필요한 변경 사항을 나타냅니다.

```
px4_add_module(
    MODULE templates__module
    MAIN module
```
```
    COMPILE_FLAGS
        -DDEBUG_BUILD
```
```
    SRCS
        module.cpp
    DEPENDS
        modules__uORB
    )
```

:::tip
자세한 로깅은 .cpp 파일의 맨 위에(포함하기 전에) `#define DEBUG_BUILD`를 추가하여 파일별로 활성화할 수 있습니다.
:::
