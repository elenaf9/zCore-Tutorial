# Zircon Memory Management Model
Zircon has two objects related to memory management, VMO (Virtual Memory Object) and VMAR (Virtual Memory Address Region), VMO is mainly responsible for managing physical memory pages and VMAR is mainly responsible for process virtual memory management. When a process is created and needs to use memory, it needs to create a VMO and then map this VMO to the VMAR.
