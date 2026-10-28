# The Trouble Caused by the Server's Automatic Time Sync Service


Soldiers Sortie had an outage once, and the cause was the sync service pulling the time backwards. That meant the current time could end up earlier than the timestamp of an event already recorded in memory, so subtracting one from the other produced a negative number — and since we were using DWORD, it became an enormous value and corrupted the data.  
The fix: first, it's best to turn off the time sync service on the server; then check the DWORD difference and skip the operation if it's negative. I think DWORD has to be used with care — I've run into trouble on this more than once!

