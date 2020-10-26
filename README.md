# CMake Modules for Tensorflow Lite Micro on STM32

The contents of this repository are used in several projects as a submodule:

- [stm32-tflm-cmake](https://github.com/PhilippvK/stm32-tflm-cmake)
- [stm32-tflm-hello-world](https://github.com/PhilippvK/stm32-tflm-cmake)
- [stm32-tflm-mirco-speech](https://github.com/PhilippvK/stm32-tflm-cmake)
- [stm32-tflm-mnist](https://github.com/PhilippvK/stm32-tflm-cmake)

## Disclaimer

The original source of most of the modules can be found here: [bKo/stm32-cmake](https://github.com/ObKo/stm32-cmake)

The tensorflow and FATFS functionality was initially added by [@alxhoff](https://github.com/alxhoff) and extended with CMSIS-NN and a few other features by [@PhilippvK](https://github.com/PhilippvK).

## Features

- Many STM32 Boards/Chips/Families supported
- Supports STM32 HAL/LL/BSP Drivers
- USBHost/USB-Device Support
- FATFS Extension Support
- Supports Tensorflow Lite
- Includes optional CMSIS-NN extensions
- (`FreeRTOS Support`)

## Details on Patches and Workarounds

A few Tensorflow-related patches are included in the Modules to get it running on the STM32 Boards. They reasons why they exist are explained here:

- `FindCMSISNN.cmake`

  Missing Declarartion of `SXTB16_RORn` in [CMSIS-NN](https://github.com/ARM-software/CMSIS_5)
  
  See https://github.com/ARMmbed/mbed-os/issues/12568)
    
  **Workaround:** (Likely to break if original sources line numbers changes)
  
  ```
  MESSAGE(STATUS "Adding Workaround for '__SXTB16_RORn' to CMSIS-NN...")
  EXECUTE_PROCESS(COMMAND "bash" "-c" "grep -rq '__patched_SXTB16_RORn' ${ARM_CMSIS_DIR}/NN/Source/NNSupportFunctions/arm_nn_mat_mult_nt_t_s8.c || sed -i -E 's@__SXTB16_RORn@__patched_SXTB16_RORn@g' ${ARM_CMSIS_DIR}/NN/Source/NNSupportFunctions/arm_nn_mat_mult_nt_t_s8.c")
  EXECUTE_PROCESS(COMMAND "bash" "-c" "grep -rq 'uint32_t __patched_SXTB16_RORn' ${ARM_CMSIS_DIR}/NN/Source/NNSupportFunctions/arm_nn_mat_mult_nt_t_s8.c || sed -i -E $'35 a \\\n\\\n// Work around for https://github.com/ARMmbed/mbed-os/issues/12568\\\n__STATIC_FORCEINLINE uint32_t __patched_SXTB16_RORn(uint32_t op1, uint32_t rotate) {\\\n  uint32_t result;\\\n  __ASM (\"sxtb16 %0, %1, ROR %2\" : \"=r\" (result) : \"r\" (op1), \"i\" (rotate) );\\\n  return result;\\\n}' ${ARM_CMSIS_DIR}/NN/Source/NNSupportFunctions/arm_nn_mat_mult_nt_t_s8.c")
  ```

- `FindTFLite.cmake`

  - Get rid of duplicate declarations of CMSIS-NN kernels:
  
    **Workaround:** Remove the TFLM-version of the kernels is CMSIS-NN is enabled
    
    ```
    FILE(GLOB TFL_KERNELS_CMSISNN_SRCS
      ${TFLM_SRC}/kernels/cmsis-nn/*.cc
    )

    FOREACH(src ${TFL_KERNELS_CMSISNN_SRCS})
      GET_FILENAME_COMPONENT(src_name ${src} NAME)
      SET(src_path "${TFLM_SRC}/kernels/${src_name}")
      LIST(FIND TFL_KERNELS_SRCS ${src_path} TFL_KERNELS_SRCS_FOUND_INDEX)
      IF(${TFL_KERNELS_SRCS_FOUND_INDEX} GREATER_EQUAL 0)
        MESSAGE(STATUS "Replacing TFLM version of ${src_name} by CMSIS-NN variant...")
        LIST(REMOVE_ITEM TFL_KERNELS_SRCS ${src_path})
        LIST(APPEND TFL_KERNELS_SRCS ${src})
      ENDIF()
    ENDFOREACH()
    ```
    
  - Remove Prefix (`cmsis/CMSIS/NN/Include`) of [CMSIS-NN](https://github.com/ARM-software/CMSIS_5) Includes to confrom to TF's `#include`-"Policy" 
  
    **Workaround:**
    
    ```
    EXECUTE_PROCESS(COMMAND bash "-c" "grep -qri 'cmsis/CMSIS/NN/Include' ${TFLM_SRC}/kernels/cmsis-nn && find ${TFLM_SRC}/kernels/cmsis-nn -iname '*.cc' | xargs sed -i -E $'s@cmsis/CMSIS/NN/Include/@@g' || :")
    ```
  
  
