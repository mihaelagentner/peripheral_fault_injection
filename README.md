# How to Design Fault Injection Tests for Embedded Safety-Critical Systems

Fault injection is one of the most effective ways to verify that an embedded system responds correctly when hardware or software faults occur. In safety-critical systems, it is not enough to detect faults during normal testing. Engineers must deliberately introduce abnormal conditions and confirm that diagnostic mechanisms behave as intended.

A well-designed fault injection strategy helps answer questions such as:

- Does the system detect the fault?
- Is the fault reported correctly?
- Does the system transition to the expected safe state?
- Is the diagnostic coverage sufficient?
- Can the fault be isolated to the correct component or subsystem?
- Does recovery occur correctly after the fault is removed?

The goal is not simply to “break” the system. The purpose is to validate that safety mechanisms operate predictably under controlled fault conditions.

## 1. Start With the Safety Mechanism, Not the Fault

A common mistake is to begin by asking, “What faults can I inject?”

A better approach is to start with the safety requirement or diagnostic mechanism being tested.

For each mechanism, identify:

- the monitored hardware or software element
- the fault condition that must be detected
- the expected detection method
- the expected response
- the maximum acceptable detection time
- the expected recovery behavior

For example, if the system includes ECC-protected Flash memory, the safety mechanism may require detection of single-bit or multi-bit memory corruption.

The test should therefore be built around validating that specific mechanism rather than randomly corrupting memory.

A useful structure is:

```text
Safety Requirement
        ↓
Diagnostic Mechanism
        ↓
Injectable Fault
        ↓
Expected Detection
        ↓
Expected System Response
```

This keeps the fault injection test traceable to an actual requirement.

## 2. Define the Expected Result Before Injecting the Fault

Every fault injection test should have a clearly defined expected result.

For example:

```text
Fault:
Corrupt a Flash memory location protected by ECC.

Expected behavior:
1. ECC logic detects the corruption.
2. The appropriate fault status is set.
3. Diagnostic software identifies the error.
4. The system reports or logs the fault.
5. The system performs the required safety reaction.
```

Without a defined expected result, a fault injection test can easily become exploratory debugging instead of verification.

The expected reaction should ideally come directly from a software requirement or safety requirement.

## 3. Use Multiple Fault Injection Techniques

There is no single fault injection method that works for every peripheral.

Depending on the target, faults can be introduced through:

- debugger memory modification
- register manipulation
- software hooks
- forced peripheral states
- communication corruption
- invalid configuration
- signal manipulation
- timing changes
- hardware fault injection

Debugger-assisted fault injection is especially useful during embedded development because it allows controlled manipulation of memory and registers without modifying production hardware.

Tools such as JTAG debuggers can be used to:

- modify RAM
- alter peripheral registers
- corrupt variables
- force error flags
- pause execution at specific points
- inspect diagnostic state after injection

This provides a repeatable way to introduce faults during development and verification.

## 4. SRAM Fault Injection

SRAM is a common target for diagnostic testing, especially when ECC or parity protection is used.

Possible test cases include:

- single-bit corruption
- multi-bit corruption
- corruption of safety-critical variables
- corruption of memory used by communication buffers
- corruption of stack or application data

With debugger-assisted testing, a memory location can be modified while the processor is halted.

A typical sequence might be:

```text
1. Identify an SRAM address used by the test.
2. Record the original value.
3. Halt execution.
4. Modify one or more bits.
5. Resume execution.
6. Observe the diagnostic reaction.
7. Verify that the expected fault is reported.
```

If ECC is present, it is important to understand how the device implements ECC because directly writing through a debugger may update both the data and its ECC bits.

In that case, additional device-specific techniques may be required to create an actual ECC mismatch.

## 5. Flash and ECC Fault Injection

Flash fault injection is particularly important in safety-critical systems because corrupted executable code or calibration data can create unpredictable system behavior.

Typical faults include:

- single-bit ECC errors
- double-bit ECC errors
- corrupted Flash contents
- invalid checksums
- integrity verification failures

A single-bit ECC error may be automatically corrected by hardware, while a multi-bit error may be detected but not correctable.

The test should verify both the hardware response and the software response.

For example:

```text
Single-bit error
        ↓
ECC detects error
        ↓
Hardware corrects data
        ↓
Diagnostic software records event
```

A more serious fault may result in:

```text
Multi-bit error
        ↓
ECC detects uncorrectable condition
        ↓
Fault handler executes
        ↓
System enters safe state
```

The exact response depends on the microcontroller and safety architecture.

## 6. ADC Fault Injection

Analog-to-digital converters are often used to monitor safety-related signals such as voltage, current, temperature, pressure, or optical power.

Useful injected faults include:

- value above maximum threshold
- value below minimum threshold
- stuck value
- implausible value
- rapid change beyond expected limits
- missing conversion
- conversion timeout

In many cases, ADC fault injection can be performed at the software level by substituting or overwriting the sampled value.

For example:

```c
adc_value = injected_test_value;
```

During production testing, this test hook should normally be disabled.

The important part is to verify that the monitoring logic detects the abnormal condition and responds within the required time.

## 7. UART Fault Injection

UART interfaces can fail in several ways.

Useful injected faults include:

- framing errors
- parity errors
- overrun errors
- corrupted bytes
- missing bytes
- invalid packet length
- communication timeout
- buffer overflow

Tests should verify both the peripheral-level error handling and the application-level response.

For example, if a message becomes corrupted:

```text
Corrupted UART frame
        ↓
UART error detected
        ↓
Invalid message rejected
        ↓
Error counter incremented
        ↓
System continues operating safely
```

It is often useful to combine peripheral errors with protocol errors because real communication failures may occur at either level.

## 8. CAN Fault Injection

CAN provides several built-in error detection mechanisms, making it an important area for fault injection testing.

Potential test conditions include:

- invalid CAN frames
- bus-off state
- transmit errors
- receive errors
- error counter escalation
- message timeout
- missing periodic messages
- unexpected CAN identifiers
- invalid payload values

One particularly important condition is bus-off.

The test should verify:

- detection of the bus-off condition
- application notification
- communication shutdown if required
- recovery behavior
- diagnostic logging

For safety-related communication, it is also important to test missing messages.

A system should not assume that the absence of data means the previous value remains valid indefinitely.

A timeout mechanism should mark stale data appropriately.

## 9. SPI Fault Injection

SPI communication faults can be created at both the protocol and peripheral levels.

Examples include:

- incorrect received data
- missing device response
- timeout
- corrupted register values
- invalid chip-select behavior
- unexpected peripheral status
- incorrect clock configuration
- communication retry failure

SPI fault injection is especially useful when communicating with sensors, external Flash, ADCs, or safety-related monitoring devices.

The test should verify that a communication failure does not silently propagate invalid data into the rest of the system.

For example:

```text
Sensor SPI read fails
        ↓
Driver reports error
        ↓
Application rejects sensor value
        ↓
Diagnostic state updated
        ↓
Fallback or safe behavior selected
```

## 10. Test Detection Time

A fault may be detected correctly but still fail a safety requirement if detection takes too long.

Fault injection testing should therefore measure:

```text
Fault injection time
        ↓
Fault detected
        ↓
Safety reaction initiated
```

The difference between the injection time and the detection time should be compared with the requirement.

For periodic diagnostic tasks, the detection interval often depends on the task execution period.

For example, if a diagnostic runs every 100 ms, the worst-case detection delay may approach the diagnostic period plus execution and scheduling delays.

This should be accounted for during verification.

## 11. Verify More Than the Fault Flag

A successful fault injection test should not stop at checking that an error flag was set.

The complete response should be verified.

Depending on the system, this may include:

- hardware status register
- software diagnostic flag
- diagnostic event
- error counter
- application state change
- output shutdown
- communication message
- fault log
- safe-state transition
- recovery behavior

A fault may be detected correctly at the hardware level but never propagated to the application.

That is why end-to-end verification is important.

## 12. Make Fault Injection Repeatable

Manual fault injection can be useful during development, but repeatability is essential for formal verification.

Each test should document:

- preconditions
- fault location
- injected value or condition
- exact injection procedure
- expected response
- pass/fail criteria
- recovery procedure

Where possible, debugger scripts or automated test tools should be used to reduce variation between test runs.

A repeatable test might look like:

```text
Test ID: FI_FLASH_ECC_001

Precondition:
System initialized normally.

Injection:
Introduce a single-bit Flash ECC fault.

Expected:
ECC detection occurs.

Verification:
Diagnostic event FLASH_ECC_SINGLE_BIT is reported.

Pass Criteria:
Fault detected within required interval and system remains operational.
```

## 13. Preserve Traceability

In safety-critical development, every fault injection test should trace back to a requirement.

A useful traceability chain is:

```text
Safety Goal
    ↓
Technical Safety Requirement
    ↓
Software Safety Requirement
    ↓
Diagnostic Mechanism
    ↓
Fault Injection Test
    ↓
Test Result
```

This makes it possible to demonstrate that the implemented diagnostic mechanism has actually been verified.

It also simplifies audits and safety reviews.

## 14. Avoid Testing Only the “Happy” Fault

A good test suite includes variations.

For example, instead of testing only one ADC threshold violation, test:

```text
just below threshold
exactly at threshold
just above threshold
extreme high value
extreme low value
rapid oscillation
stuck value
```

Boundary conditions often reveal problems that are not visible during normal fault testing.

The same principle applies to communication timeouts, memory corruption, and error counters.

## 15. Separate Test Mechanisms From Production Software

Software fault injection hooks can be extremely useful, but they should not accidentally remain active in production builds.

Possible approaches include:

```c
#ifdef FAULT_INJECTION_TEST
    value = injected_value;
#endif
```

or dedicated test interfaces enabled only in verification builds.

The fault injection mechanism itself should not introduce unintended behavior into the production system.

## Conclusion

Fault injection is most effective when it is treated as structured verification rather than ad-hoc debugging.

A strong fault injection strategy begins with the safety requirement, identifies the diagnostic mechanism, introduces a controlled fault, and verifies the complete system response.

For embedded systems, useful targets include SRAM, Flash and ECC, ADC, UART, CAN, SPI, communication buffers, and peripheral status registers. Debugger-assisted fault injection can make many of these tests practical and repeatable without requiring specialized hardware.

The most important question is not simply whether the system detects a fault.

It is whether the system detects the right fault, within the required time, reports it correctly, and transitions to the intended safe behavior.

That is what turns fault injection from a debugging technique into a meaningful safety-verification tool.
