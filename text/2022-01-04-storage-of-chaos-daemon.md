# Storage of chaos-daemon

## Summary

Refine the data storage of chaos-daemon.

## Background
Now the only way for chaos-daemon to get some status info is to parse RPC request from controller. 

Some examples: 

- In Stress-Chaos developers need to reply the controller with CPU-Stress PID & Mem-Stress PID to controller and controller need to send these data to chaos-daemon when it comes to recover the Stress-Chaos. 
- Network-Chaos replies nothing , so if someone apply two chaos on one pod , only the last Network-Chaos will be applied. 
When developing the recovering function of HTTP-Chaos , I find it is hard because I need to send status info from TPROXY to controller. (TPROXY -> Chaos-Daemon -> Pod-HTTP-Chaos -> HTTP-Chaos). The problem shows the framework of storing status info now makes  all sub component & Chaos-Daemon tightly coupling with the Chaos-Controller.

## Proposal
Chaos-Daemon can store data in the mount namespace of Target POD ( Suggestion From **[YangKeao](https://github.com/YangKeao)** in [#2381](https://github.com/chaos-mesh/chaos-mesh/issues/2381#issuecomment-940964249) ), and then recover the chaos or apply other chaos with reading the stored data. Provide a development guide to store the status info.

### Data storage
We can easily access to mount namespace of target POD with through `/proc/$PID/root`.If we just make `/proc/$PID/root/tmp` a position to store our status info , I think this will not bring any problem. If the target POD was deleted, the closed file will be deleted , so they have a same life time with POD. When the target POD was deleted , the opened `fd` of files inside the POD mount namespace will be delete after we closed it and we can close it with no problem.

I suggest to store data with a embeddable DB like `Badger` or `BoltDB`.

### Development guide
Data received from controller:

- Request type: Apply chaos
  - Data: // Chaos Controller transfer PID & a primary key & some chaos info like `IptablesChainsRequest` in Network Chaos.
    - Target PID
    - Primary key
    - Chaos info
  - Storage: 
    - ( Chaos info, ISAPPLIED)
    - [ Condition Backup of Middle Function , ISAPPLIED]
    - xxx
    -  Chaos Daemon apply chaos with logging some Backup info with the primary key. 
      1. Read ISAPPLIED equal to false.
      2. Applies middle function or the chaos
      3. Update ISAPPLIED to be true. 
      4. When it comes to an error, try recover the error & return error. (If recover failed ,this will be a BUG)
  - Return:
    - [Error]
- Request type: Recover chaos
  - Data: // Chaos Controller transfer PID & some chaos info like `IptablesChainsRequest` in Network Chaos.
    - Target PID
    - Primary key
  - Storage:  //  Chaos Daemon will  recover the chaos and delete the Backup info. (If recover failed ,this will be a BUG). If we use transaction to write data, the recovery will naturally be blocked by apply transaction. 
    - ( Chaos info, ISAPPLIED)
    - [ Condition Backup , ISAPPLIED]
    - xxx
  - Return:
    - [Error]

Request type:  Heart Beats

- Data: // Chaos Controller transfer PID & some chaos info like `IptablesChainsRequest` in Network Chaos.
  - Target PID
  - Primary key
- Storage:  //  Chaos Daemon will  check the Chaos info . If a middle function or the chaos is not applied , Chaos-Daemon will recover it & then apply it.
  - ( Chaos info, ISAPPLIED)
  - [ Condition Backup , ISAPPLIED]
  - xxx
- Return:
  - [Error]
