# Attack Vectors on Duress Mechanism

1. Denial of service attack on SAR.

2. If duress check is to be performed in Iso environment, it might take more time compared to other messages. When checking the recentness of the last observed block in this situation, one must count for the time required to enter and exit the Iso environment.

3. WT pretending not to receive the ACK from SAR and try to halt the ceremony.
   * WT can stop responding at will.
