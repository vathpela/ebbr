.. SPDX-License-Identifier: CC-BY-SA-4.0

****
UEFI
****

This chapter discusses specific UEFI implementation details for EBBR compliant
platforms.

UEFI Version
============
This document uses version 2.7 of the UEFI specification [UEFI]_.

UEFI Compliance
===============

EBBR compliant platforms shall conform to the requirements in [UEFI]_ § 2.6,
except where explicit exemptions are provided by this document.

EBBR compliant platforms shall also implement the UEFI services and
protocols that are listed in :ref:`appendix-uefi-requirements` of this
document.

Block device partitioning
-------------------------

The system firmware must implement support for MBR, GPT and El Torito partitioning.

UEFI System Environment and Configuration
=========================================

The resident UEFI boot-time environment shall use the highest non-secure
privilege level available.
The exact meaning of this is architecture dependent, as detailed below.

Resident UEFI firmware might target a specific privilege level.
In contrast, UEFI Loaded Images, such as third-party drivers and boot
applications, must not contain any built-in assumptions that they are to be
loaded at a given privilege level during boot time since they can, for example,
legitimately be loaded into either EL1 or EL2 on AArch64.

AArch64 Exception Levels
------------------------

UEFI shall execute as 64-bit code in AArch64 model at either EL1 or EL2,
depending on whether or not virtualization is used or supported.

UEFI Boot at EL2
^^^^^^^^^^^^^^^^

Most systems are expected to boot UEFI at EL2, to allow for the installation of
a hypervisor or a virtualization aware Operating System.

UEFI Boot at EL1
^^^^^^^^^^^^^^^^

Booting of UEFI at EL1 is most likely within a hypervisor hosted Guest
Operating System environment, to allow the subsequent booting of a
UEFI-compliant Operating System.
In this instance, the UEFI boot-time environment can be provided, as a
virtualized service, by the hypervisor and not as part of the host firmware.

UEFI Boot Services
==================

Memory Map
----------

The UEFI environment must provide a system memory map, which must include all
appropriate devices and memories that are required for booting and system
configuration.

All RAM defined by the UEFI memory map must be identity-mapped, which means
that virtual addresses must equal physical addresses.

The default RAM allocated attribute must be EFI_MEMORY_WB.

Configuration Tables
--------------------

A UEFI system that complies with this specification may provide the additional
tables via the EFI Configuration Table.

Compliant systems are required to provide one, but not both, of the following
tables.

- An Advanced Configuration and Power Interface [ACPI]_ table, or
- a Devicetree [DTSPEC]_ system description

As stated above, EBBR systems must not provide both ACPI and Devicetree
tables at the same time.
Systems that support both interfaces must provide a configuration
mechanism to select either ACPI or Devicetree,
and must ensure only the selected interface is provided to the OS loader.

UEFI Secure Boot (Optional)
---------------------------

UEFI Secure Boot is optional for this specification.

If Secure Boot is implemented, it must conform to the UEFI specification for Secure Boot. There are no additional
requirements for Secure Boot.

UEFI Runtime Services
=====================

UEFI runtime services exist after the call to ExitBootServices() and are
designed to provide a limited set of persistent services to the platform
Operating System or hypervisor.
Functions contained in EFI_RUNTIME_SERVICES are expected to be available
during both boot services and runtime services.
However, it isn't always practical for all EFI_RUNTIME_SERVICES functions
to be callable during runtime services due to hardware limitations.
If any EFI_RUNTIME_SERVICES functions are only available during boot services
then firmware shall provide the global `RuntimeServicesAvailable` variable to
indicate which functions are available during runtime services.
Functions that are not available during runtime services shall return
EFI_UNSUPPORTED.

Table :numref:_uefi_runtime_service_requirements details which EFI_RUNTIME_SERVICES
are required to be implemented during boot services and runtime services.

.. _uefi_runtime_service_requirements:
.. table:: EFI_RUNTIME_SERVICES Implementation Requirements

   ============================== ============= ================
   EFI_RUNTIME_SERVICES function  Boot Services Runtime Services
   ============================== ============= ================
   EFI_GET_TIME                   Optional      Optional
   EFI_SET_TIME                   Optional      Optional
   EFI_GET_WAKEUP_TIME            Optional      Optional
   EFI_SET_WAKEUP_TIME            Optional      Optional
   EFI_SET_VIRTUAL_ADDRESS_MAP    N/A           Required
   EFI_CONVERT_POINTER            N/A           Required
   EFI_GET_VARIABLE               Required      Optional
   EFI_GET_NEXT_VARIABLE_NAME     Required      Optional
   EFI_SET_VARIABLE               Required      Optional
   EFI_GET_NEXT_HIGH_MONO_COUNT   N/A           Optional
   EFI_RESET_SYSTEM               Required      Optional
   EFI_UPDATE_CAPSULE             Optional      Optional
   EFI_QUERY_CAPSULE_CAPABILITIES Optional      Optional
   EFI_QUERY_VARIABLE_INFO        Optional      Optional
   ============================== ============= ================

Runtime Device Mappings
-----------------------

Firmware shall not create runtime mappings, or perform any runtime IO that will
conflict with device access by the OS.
Normally this means a device may be controlled by firmware, or controlled by
the OS, but not both.
e.g. If firmware attempts to access an eMMC device at runtime then it will
conflict with transactions being performed by the OS.

Devices that are provided to the OS (i.e., via PCIe discovery or ACPI/DT
description) shall not be accessed by firmware at runtime.
Similarly, devices retained by firmware (i.e., not discoverable by the OS)
shall not be accessed by the OS.

Only devices that explicitly support concurrent access by both firmware and an
OS may be mapped at runtime by both firmware and the OS.

Real-time Clock (RTC)
^^^^^^^^^^^^^^^^^^^^^

Not all embedded systems include an RTC, and even if one is present,
it may not be possible to access the RTC from runtime services.
e.g., The RTC may be on a shared I2C bus which runtime services cannot access
because it will conflict with the OS.

If firmware does not support access to the RTC, then GetTime() and
SetTime() shall return EFI_UNSUPPORTED,
and the OS must use a device driver to control the RTC.

UEFI Reset and Shutdown
-----------------------

The UEFI Runtime service ResetSystem() must implement the following commands,
for purposes of power management and system control.

- EfiResetCold()
- EfiResetShutdown()
  * EfiResetShutdown must not reboot the system.

If firmware updates are supported through the Runtime Service of
UpdateCapsule(), then ResetSystem() might need to support the following
command:

- EfiWarmReset()

.. note:: On platforms implementing the Power State Coordination Interface
   specification [PSCI]_, it is still required that EBBR compliant
   Operating Systems calls to reset the system will go via Runtime Services
   and not directly to PSCI.

Runtime Variable Access
-----------------------

There are many platforms where it is difficult to implement SetVariable() for
non-volatile variables during runtime services because the firmware cannot
access storage after ExitBootServices() is called.

e.g., If firmware accesses an eMMC device directly at runtime, it will
collide with transactions initiated by the OS.
Neither U-Boot nor Tianocore have a generic solution for accessing or updating
variables stored on shared media. [#OPTEESupplicant]_

If a platform does not implement modifying non-volatile variables with
SetVariable() after ExitBootServices(), then it must implement support for
discovering this during Boot Services via the "RuntimeServicesSupported"
variable (see UEFI Mantis 1961).

Such a system may also support exporting the variable storage to the
kernel via a UEFI configuration table and re-loading it from a Capsule on
Disk as described in [UEFI]_ 8.5.5.  If this is supported, the platform
must implement the following:

- The firmware must provide BS variable named "CapsuleVariableSupport"
  under the VARIABLE_STORAGE_GUID, with the following bits defined:

    #define EBBR_CAPSULE_VARIABLE_EXPORT 	   0x01
    #define EBBR_CAPSULE_VARIABLE_IMPORT 	   0x02
    #define EBBR_CAPSULE_VARIABLE_IMPORT_AUTH  	   0x04
    #define EBBR_CAPSULE_VARIABLE_IMPORT_AUTH_2    0x08
    #define EBBR_CAPSULE_VARIABLE_IMPORT_AUTH_3    0x10

- If the firmware supports exporting variables via a configuration table,
  the EBBR_CAPSULE_VARIABLE_EXPORT bit must be set, and the firmware must
  create a configuration table identified by the VARIABLE_STORAGE_GUID
  (defined below) before any EFI applications are started, and must update it
  any time a variable is altered via SetVariable(), until ExitBootServices()
  has successfully returned.
- If the firmware supports importing non-volatile variables via a Capsule
  on Disk, the EBBR_CAPSULE_VARIABLE_IMPORT bit must be set, and the
  firmware must load its initial variable storage during boot services
  from a capsule with the EFI_CAPSULE_HEADER.CapsuleGuid set to
  VARIABLE_STORAGE_GUID.
- If the firmware supports authenticated variables, the bits
  EBBR_CAPSULE_VARIABLE_IMPORT_AUTH, EBBR_CAPSULE_VARIABLE_IMPORT_AUTH_2,
  and EBBR_CAPSULE_VARIABLE_IMPORT_AUTH_3 must be set to indicate support for
  EFI_VARIABLE_AUTHENTICATION, EFI_VARIABLE_AUTHENTICATION_2, and
  EFI_VARIABLE_AUTHENTICATION_3 descriptors defined in [UEFI]_ 8.2.
- If the CapsuleVariableSupport variable is not set, the OS must behave as if
  all bits are 0.
- Any bits which are not present in the CapsuleVariableSupport variable must
  be treated as 0.

Variable storage data format
----------------------------

The Variable Configuration Table and the Variable Update Capsule structure
share the same data format, and are structured as a capsule update containing
a packed array of update records:

#define VARIABLE_STORAGE_GUID \
        {0x1a3fb419, 0x2171, 0x458d,\
         {0xb8, 0xb4, 0xbe, 0xa3, 0x0c, 0x9f, 0x6b, 0xab }}

typedef struct {
  CHAR16[64]   VariableName;
  EFI_GUID     VendorGuid;
  UINT32       Attributes;
  UINT32       DataSize;
  UINT8[]      Data;
} EBBR_VARIABLE;

typedef struct {
  EFI_CAPSULE_HEADER                                     Header;
  UINT8[Header.HeaderSize - sizeof(EFI_CAPSULE_HEADER)]  Reserved;
  EBBR_VARIABLE[]                                        Variables;
} EBBR_VARIABLE_BUNDLE __attribute__((__packed__));

- EBBR_VARIABLE_BUNDLE.Header.CapsuleGuid must be VARIABLE_STORAGE_GUID
- EBBR_VARIABLE_BUNDLE.Reserved may be 0 or more bytes.
- EBBR_VARIABLE_BUNDLE.Header.HeaderSize is equal to the starting offset of
  the EBBR_VARIABLE_BUNDLE.Variables array.
- EBBR_VARIABLE_BUNDLE.Variables may be 0 or more bytes.
- EBBR_VARIABLE_BUNDLE.Header.CapsuleImageSize is the full size of the capsule
  including the Header, Reserved, and all Variable array members.
- EBBR_VARIABLE_BUNDLE.Header.Flags must not have any of the following set:
  CAPSULE_FLAGS_INITIATE_RESET
- EBBR_VARIABLE_BUNDLE.Header.Flags should have all of the following set:
  CAPSULE_FLAGS_PERSIST_ACROSS_RESET
  CAPSULE_FLAGS_POPULATE_SYSTEM_TABLE

Variable Configuration Table creation
-------------------------------------

Any platform firmware which supports EBBR_CAPSULE_VARIABLE_EXPORT must
install a UEFI Configuration Table with all appropriate variables specified
in [UEFI]_ 3.3, as well as any variables which have been imported from a
Variable Update Capsule or set through SetVariable().

- The platform firmware must not include multiple entries for the same
  variable.
- The platform should avoid storing any secrets in variables, including
  variables without EFI_VARIABLE_RUNTIME_SERVICES set.

Variable Configuration Table processing
---------------------------------------

When processing the Variable Configuration Table, the OS must treat each
EBBR_VARIABLE_BUNDLE.Variable entry as if it were a call to SetVariable()
before ExitBootServices() has been called:

- The variables are processed according to the requirements in [UEFI]_ 8.2
- Any update which would result in SetVariable() returning an error must
  be ignored.
- The OS should preserve entries with EFI_VARIABLE_NON_VOLATILE set but
  EFI_VARIABLE_RUNTIME_SERVICES unset, and save them to a Variable Updates
  Capsule before system reset.
- Any entry without EFI_VARIABLE_RUNTIME_SERVICES set must not be exposed to
  consumers of GetVariable().
- Any entry authenticated with an Authentication Descriptor the OS does not
  support should be preserved, but must not be exposed to consumers of
  GetVariable().  Any following entry for any such variable must be treated
  the same.
- All authenticated variables should have their Authentication Descriptors
  preserved, but only the encapsulated data should be presented through
  GetVariable()-like interfaces.
- The OS must take measures to prevent data in variables without
  EFI_VARIABLE_RUNTIME_SERVICES set from being exposed to unprivileged tasks.

Runtime Processing
------------------

The OS must take certain measures during runtime operation to insure
consistency:

- The OS must keep a log of any updates to variables, including delete,
  append, and create operations.
- The OS should attempt to simulate each operation as it would be applied
  during the Variable Updates Capsule processing, in order to maintain a
  view which is coherent across a reset.

Variable Updates Capsule creation
---------------------------------

Before system reset, the OS should create a Variable Updates Capsule as a
Capsule on Disk defined in [UEFI]_ 8.5.5 .  If a platform supports both
EBBR_CAPSULE_VARIABLE_EXPORT and EBBR_CAPSULE_VARIABLE_IMPORT, the capsule
should preserve the following from the Variable Configuration Table, if it
was present:

- EBBR_VARIABLE_BUNDLE.Header.HeaderSize, EBBR_VARIABLE_BUNDLE.Header.Flags,
  and EBBR_VARIABLE_BUNDLE.Reserved fields
- Any variables with EFI_VARIABLE_NON_VOLATILE set, including those without
  EFI_VARIABLE_RUNTIME_SERVICES set
- Any variables authenticated with Authentication Descriptors not supported
  by the OS.
- Runtime updates to authenticated variables must be included individually,
  including any authenticated deletion.
- Runtime operations on newly created variables which are not authenticated
  may be coalesced to a single entry.

Variable Updates Capsule processing
-----------------------------------

During boot, the system firmware must create the variables specified in [UEFI]_
3.3 before processing the capsule update, and it must ensure that the variables
implemented in the UEFI spec are treated as specified at all times.  When
processing a Variable Updates Capsule, the firmware must process each record in
the order they appear in the Variables array as if each were a call to
SetVariable() after ExitBootServices():

- The variables are processed according to the requirements in [UEFI]_ 8.2
- Any update without the EFI_VARIABLE_NON_VOLATILE attribute set must be
  ignored.
- Any update which would result in SetVariable() returning an error must
  be ignored.
- Any variable authenticated with an unsupported Authentication Descriptor
  not supported by the platform must be ignored.
