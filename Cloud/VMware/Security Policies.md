## Security
### Promiscuous mode
- **Reject**. The VM network adapter receives only frames that are addressed to the virtual machine.
- **Accept**.The virtual switch forwards all frames to the virtual machine in compliance with the active VLAN policy for the port to which the VM network adapter is connected.

**Note:** Promiscuous mode is insecure mode of operation. Firewalls, port scanners, intrusion detection systems, must run in promiscuous mode.
### MAC address changes
- **Reject**. If the guest OS changes the effective MAC address of the virtual machine to a value that is different from the MAC address of the VM network adapter, the switch drops all inbound frames to the adapter.
- If the guest OS changes the effective MAC address of the virtual machine back to the MAC address of the VM network adapter, the virtual machine receives frames again.
- **Accept**. If the guest OS changes the effective MAC address of the virtual machine to a value that is different from the MAC address of the VM network adapter, the switch allows frames to the new address to pass.
### Forged transmits
- **Reject**. The switch drops any outbound frame from a virtual machine adapter with a source MAC address that is different from the one in the .vmx configuration file.
- **Accept**. The switch does not perform filtering, and permits all outbound frames.