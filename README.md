# Refcount-Overflow-Use-After-Free
The join_session_keyring function in security/keys/process_keys.c in the Linux kernel before 4.4.1 mishandles object references in a certain error case, which allows local users to gain privileges or cause a denial of service (integer overflow and use-after-free) via crafted keyctl commands.

Ubuntu Version-14.04.1
Kernel Version-3.18.25

Each process can create a keyring for the current session using keyctl(KEYCTL_JOIN_SESSION_KEYRING, name)  and can choose to either assign a name to the keyring or not by passing NULL. The keyring object can be shared between processes by referencing the same keyring name. If a process already has a session keyring, this same system call will replace its keyring with a new one. If an object is shared between processes, the object’s internal refcount, stored in a field called usage, is incremented. The leak occurs when a process tries to replace its current session keyring with the very same one. As we see in the code snippet, taken from kernel version 3.18, the execution jumps to error2 label which skips the call to key_put and leaks the reference that was increased by find_keyring_by_name.

Even though the bug itself can directly cause a memory leak, it has far more serious consequences. After a quick examination of the relevant code flow, we found that the usage field used to store the reference count for the object is of type atomic_t, which under the hood, is basically an int – meaning 32-bit on both 32-bit and 64-bit architectures. While every integer is theoretically possible to overflow, this particular observation makes practical exploitation of this bug as a way to overflow the reference count seem feasible. And it turns out no checks are performed to prevent overflowing the usage field from wrapping around to 0.

If a process causes the kernel to leak 0x100000000 references to the same object, it can later cause the kernel to think the object is no longer referenced and consequently free the object. If the same process holds another legitimate reference and uses it after the kernel freed the object, it will cause the kernel to reference deallocated, or a reallocated memory. This way, we can achieve a use-after-free, by using the exact same bug from before. 
The outline of the steps that to be executed by the exploit code is as follows:

1.Hold a (legitimate) reference to a key object

2.Overflow the same object’s usage

3.Get the keyring object freed

4.Allocate a different kernel object from user-space, with a user-controlled content, over the same memory previously used by the freed keyring object

5.Use the reference to the old key object and trigger code execution

Video of the exploit:https://drive.google.com/file/d/1Iu_--ifK2GQXrlIS8DmyJfQiM40xox93/view?usp=sharing


