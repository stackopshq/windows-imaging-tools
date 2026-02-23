# Implementation Plan: QEMU Guest Agent Checksum Verification

## Overview

This implementation plan breaks down the QEMU Guest Agent checksum verification feature into discrete coding tasks. The feature adds SHA256 checksum verification to the QEMU Guest Agent installation process while maintaining full backward compatibility with existing configurations.

The implementation follows a priority-based configuration system where a new `[virtio_qemu_guest_agent]` section takes precedence over the legacy `install_qemu_ga` parameter. Core functionality includes checksum computation using .NET Framework's SHA256 implementation, case-insensitive verification, and automatic cleanup on verification failure.

## Tasks

- [x] 1. Extend configuration parsing to support new section
  - [x] 1.1 Add configuration options for [virtio_qemu_guest_agent] section
    - Add `url` and `checksum` parameters to `Get-AvailableConfigOptions` in Config.psm1
    - Include descriptions and examples for each parameter
    - Add validation regex for checksum format: `^[a-fA-F0-9]{64}$`
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5_
  
  - [ ]* 1.2 Write property test for configuration section recognition
    - **Property 1: Configuration Section Recognition**
    - **Validates: Requirements 1.1, 1.2, 1.3**
    - Generate random URLs and checksums, verify parser extracts them correctly
  
  - [ ]* 1.3 Write property test for checksum format validation
    - **Property 2: Checksum Format Validation**
    - **Validates: Requirements 1.4, 1.5**
    - Generate random strings, verify only valid 64-char hex strings are accepted
  
  - [ ]* 1.4 Write unit tests for configuration parsing
    - Test parsing with both url and checksum
    - Test parsing with url only
    - Test rejection of invalid checksum formats
    - Test rejection of empty url
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5_

- [x] 2. Implement checksum computation function
  - [x] 2.1 Create Get-FileSHA256Checksum function in WinImageBuilder.psm1
    - Implement using System.Security.Cryptography.SHA256
    - Use FileStream for memory-efficient reading
    - Return lowercase hexadecimal string
    - Properly dispose of streams and crypto objects
    - Throw error if file doesn't exist
    - _Requirements: 2.1_
  
  - [ ]* 2.2 Write property test for checksum computation idempotence
    - **Property 3: Checksum Computation Idempotence**
    - **Validates: Requirements 2.1, 2.6**
    - Generate random temp files, verify computing checksum multiple times produces identical results
  
  - [ ]* 2.3 Write unit tests for checksum computation
    - Test with known file and expected checksum
    - Test error handling for non-existent file
    - Test with various file sizes
    - _Requirements: 2.1_

- [x] 3. Implement checksum verification function
  - [x] 3.1 Create Test-FileChecksum function in WinImageBuilder.psm1
    - Call Get-FileSHA256Checksum to compute actual checksum
    - Perform case-insensitive comparison with expected checksum
    - Log both checksums on mismatch
    - Delete file on verification failure
    - Throw descriptive error on mismatch
    - Log success message on match
    - _Requirements: 2.2, 2.3, 2.4, 2.5_
  
  - [ ]* 3.2 Write property test for checksum verification success
    - **Property 4: Checksum Verification Success**
    - **Validates: Requirements 2.2, 2.4**
    - Generate random files, compute checksums with varied case, verify all pass
  
  - [ ]* 3.3 Write property test for checksum verification failure and cleanup
    - **Property 5: Checksum Verification Failure and Cleanup**
    - **Validates: Requirements 2.3**
    - Generate random files with wrong checksums, verify file deletion and error throwing
  
  - [ ]* 3.4 Write property test for case insensitivity
    - **Property 6 (from requirements): Case Insensitivity Property**
    - **Validates: Requirements 2.4**
    - Generate checksums in various cases, verify all match correctly
  
  - [ ]* 3.5 Write unit tests for checksum verification
    - Test matching checksum (lowercase)
    - Test matching checksum (uppercase)
    - Test matching checksum (mixed case)
    - Test mismatched checksum
    - Verify file deletion on mismatch
    - Verify error message includes both checksums
    - _Requirements: 2.2, 2.3, 2.4, 2.5_

- [ ] 4. Checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [x] 5. Implement configuration resolution function
  - [x] 5.1 Create Get-QemuGuestAgentConfig function in WinImageBuilder.psm1
    - Accept Config hashtable and OsArch parameters
    - Check for [virtio_qemu_guest_agent] section url parameter (highest priority)
    - If present, return url and checksum with Install=$true
    - If not, check install_qemu_ga parameter (legacy)
    - If 'True', generate default Fedora URL based on OsArch
    - If custom URL, return that URL
    - If 'False' or empty, return Install=$false
    - Return hashtable with Url, Checksum, and Install keys
    - Log which configuration source is being used
    - Validate URL format (starts with http:// or https://)
    - Warn if checksum provided without URL
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 8.2, 8.3, 8.4, 8.5_
  
  - [ ]* 5.2 Write property test for new configuration section priority
    - **Property 6: New Configuration Section Priority**
    - **Validates: Requirements 3.1, 3.2**
    - Generate configs with both new and legacy sections, verify new section always wins
  
  - [ ]* 5.3 Write property test for legacy configuration fallback
    - **Property 7: Legacy Configuration Fallback**
    - **Validates: Requirements 3.4**
    - Generate configs with only legacy parameters, verify correct URL resolution
  
  - [ ]* 5.4 Write property test for configuration resolution determinism
    - **Property 8: Configuration Resolution Determinism**
    - **Validates: Requirements 3.7**
    - Generate random configs, resolve multiple times, verify identical results
  
  - [ ]* 5.5 Write property test for backward compatibility invariant
    - **Property 9: Backward Compatibility Invariant**
    - **Validates: Requirements 4.6**
    - Generate legacy configs, verify behavior matches pre-feature behavior
  
  - [ ]* 5.6 Write unit tests for configuration resolution
    - Test new section with both url and checksum
    - Test new section with url only
    - Test legacy True (default URL)
    - Test legacy custom URL
    - Test legacy False (skip installation)
    - Test no configuration (skip installation)
    - Test new section priority over legacy True
    - Test new section priority over legacy custom URL
    - Test empty URL error
    - Test checksum without URL warning
    - Test invalid URL format error
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 8.2, 8.3, 8.4, 8.5_

- [x] 6. Modify Download-QemuGuestAgent function
  - [x] 6.1 Update function signature and implementation
    - Change parameter from [string]$QemuGuestAgentConfig to [hashtable]$Config
    - Call Get-QemuGuestAgentConfig to resolve configuration
    - Check Install flag and return early if false
    - Extract Url and Checksum from resolved config
    - Log whether checksum verification will be performed
    - Maintain existing download logic with Execute-Retry
    - Call Test-FileChecksum if checksum is provided
    - Log installation source and checksum status
    - _Requirements: 5.1, 5.2, 5.4, 5.5_
  
  - [ ]* 6.2 Write property test for download without verification
    - **Property 10: Download Without Verification**
    - **Validates: Requirements 5.1**
    - Mock downloads with no checksum, verify no verification is called
  
  - [ ]* 6.3 Write unit tests for modified download function
    - Test download with checksum (mock network call)
    - Test download without checksum (mock network call)
    - Test checksum verification is called when checksum provided
    - Test checksum verification is skipped when no checksum
    - Test installation skipped when Install=false
    - _Requirements: 5.1, 5.2, 5.4_

- [ ] 7. Update all callers of Download-QemuGuestAgent
  - [-] 7.1 Find and update all function calls
    - Search for all calls to Download-QemuGuestAgent in WinImageBuilder.psm1
    - Change from passing $config.install_qemu_ga to passing $config
    - Verify all callers pass the full config hashtable
    - _Requirements: 4.1, 4.2, 4.3, 4.4_
  
  - [ ]* 7.2 Write integration tests for end-to-end flows
    - Test end-to-end with new section and checksum
    - Test end-to-end with new section without checksum
    - Test end-to-end with legacy True
    - Test end-to-end with legacy custom URL
    - Test end-to-end with legacy False
    - _Requirements: 4.1, 4.2, 4.3, 4.4_

- [ ] 8. Checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 9. Add error handling and logging
  - [ ] 9.1 Implement comprehensive error handling
    - Add error handling for invalid checksum format in config parsing
    - Add error handling for empty URL in Get-QemuGuestAgentConfig
    - Add error handling for invalid URL format
    - Add error handling for download failures
    - Add error handling for checksum computation failures
    - Ensure all error messages are descriptive and actionable
    - _Requirements: 6.1, 6.2, 6.3, 6.6_
  
  - [ ] 9.2 Add comprehensive logging
    - Log configuration source being used (new section vs legacy)
    - Log whether checksum verification will be performed
    - Log checksum verification success with hash value
    - Log checksum verification failure with expected and actual hashes
    - Log download URL and status
    - Log installation skip reasons
    - _Requirements: 5.5, 6.1, 6.2, 6.4, 6.5_
  
  - [ ]* 9.3 Write unit tests for error handling
    - Test invalid checksum format error
    - Test empty URL error
    - Test invalid URL format error
    - Test checksum without URL warning
    - Test download failure error
    - Test checksum mismatch error
    - Verify error messages are descriptive
    - _Requirements: 6.1, 6.2, 6.3, 6.6_

- [ ] 10. Update documentation
  - [ ] 10.1 Update README.md
    - Add section explaining checksum verification feature
    - Include example configuration with new section
    - Explain how to obtain SHA256 checksums
    - Explain security benefits
    - Clarify backward compatibility
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.5, 7.6_
  
  - [ ] 10.2 Update configuration example file
    - Add commented example of [virtio_qemu_guest_agent] section to Examples/windows-image-config-example.ini
    - Include both url and checksum parameters
    - Add comments explaining each parameter
    - _Requirements: 7.1, 7.4_
  
  - [ ] 10.3 Add inline code comments
    - Add comment-based help for Get-FileSHA256Checksum
    - Add comment-based help for Test-FileChecksum
    - Add comment-based help for Get-QemuGuestAgentConfig
    - Add comments explaining priority system logic in Get-QemuGuestAgentConfig
    - Add comments explaining checksum verification flow in Download-QemuGuestAgent
    - _Requirements: 7.2_

- [ ] 11. Create test data generators for property-based tests
  - [ ] 11.1 Implement test data generators
    - Create Generate-RandomUrl function
    - Create Generate-RandomChecksum function
    - Create Generate-RandomString function
    - Create Create-RandomTempFile function
    - Create Vary-Case function for case variation testing
    - Create Create-ConfigWithSection helper
    - Create Create-ConfigWithBoth helper
    - Create Create-ConfigWithLegacy helper
    - Create Create-RandomConfig helper
    - Create Create-LegacyConfig helper
  
  - [ ]* 11.2 Write unit tests for test generators
    - Test Generate-RandomUrl produces valid URLs
    - Test Generate-RandomChecksum produces 64-char hex strings
    - Test Create-RandomTempFile creates valid files
    - Test Vary-Case produces varied case strings

- [ ] 12. Final checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation
- Property tests validate universal correctness properties from the design document
- Unit tests validate specific examples and edge cases
- The implementation uses PowerShell with .NET Framework for cryptographic operations
- All existing configurations must continue to work without modification (backward compatibility)
- Configuration resolution follows a clear priority system: new section > legacy parameter
- Checksum verification is mandatory when a checksum is provided (no bypass)
- Failed verification results in immediate file deletion and installation abort
