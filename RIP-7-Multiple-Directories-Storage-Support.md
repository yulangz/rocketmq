# Status
Current State: Proposed     
Authors: Jason918     
Shepherds: [author](github_link)     
Mailing List discussion: <apache mailing list archive>     
Pull Request: #PR_NUMBER     
Released: <relased_version>    
  
# Background & Motivation
## What do we need to do
## Will we add a new module? 
No
## Will we add new APIs?
No
## Will we add new feature?
Yes. The commitLog can be stored in multiple directories, and the directories should be on different disks. 
# Why should we do that
## Are there any problems of our current project?
Yes, In our production environment, some of our machine have multiple disks (e.g. 12 disks with 5TB storage capacity each).  We can’t make the most of the storage with current version of RocketMQ.
## What can we benefit proposed changes?
Improve the disk usage for machines with multiple disks.

# Goals
## What problem is this proposal designed to solve?
The commitLog files on the broker can only be stored in one disk.
## To what degree should we solve the problem?
For simplicity , we can assume the storage capacity for each directory is the same. And we can allocate files in a round-robin manner.
# Non-Goals
## What problem is this proposal NOT designed to solve?
Nothing specific.
## Are there any limits of this proposal?
The parameter diskMaxUsedSpaceRatio still can only set with one value, which means you can not set different maximum used space ratio for different disk. 
# Changes
# Architecture

This solution I proposed consists of two operations: write and delete.    
## 1.  Write
The commit log file path is determined in the method  org.apache.rocketmq.store.MappedFileQueue#getLastMappedFile(long, boolean) .
     
We can add property for the list of storePathes, and use one path for each commitLog file. The path chosen strategy can be simply implemented by the hash of createOffset. The file storage difference is shown in the following figures.      


## 2. Delete
Commitlog file deletion is handled in the class       org.apache.rocketmq.store.DefaultMessageStore.CleanCommitLogService.
Once we introduced multiple disk support for commitLog store paths, for simplicity, we can assume the storage capacity we can use is the same. So the deletion should be executed when any of the disks reached the threshold (set by diskMaxUsedSpaceRatio).    
Interface Design/Change      
Config: expanding the notion of storePathCommitLog by add ‘:’ as the separator for different directories, such as 
‘/data0/store:/data1/store:/data2/store’      
MappedFileQueue        
Add a property :      
List<String> storePaths;       
Constructor:      
Parse storePath and create storePaths if it contains ‘:’      
Method load()     
List all files in the storePaths.
Method destroy()
Delete all storePaths.       
Method getLastMappedFile      
Choose a store path from storePaths by hash the createOffset         
CommitLogService        
isSpaceToDelete       
Iterate all storePaths and return true if any of the paths is full.      
# Compatibility, Deprecation, and Migration Plan
Are backward and forward compatibility taken into consideration?
The storePath can not be changed for one broker instance.
## Are there deprecated APIs?
no
## How do we do migration?
If we want to change the storePath config, we can run another broker instance on the same machine. And set the old broker instance as read only, and new data should all be written into new broker.  When the commitLog data got all expired, we can shutdown the old one for good.       
# Implementation Outline
We will implement the proposed changes by 1 phases.        
Add config       
Implement write        
Implement delete      
Rejected Alternatives        
Nothing yet.       
