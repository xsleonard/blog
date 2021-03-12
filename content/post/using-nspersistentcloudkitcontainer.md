+++
title = "Using NSPersistentCloudKitContainer"
tags = ["ios", "mobile development"]
date = "2020-07-01T00:00:00+00:00"
draft = true
+++

## Create the app with CloudKit enabled or add it afterwards
- Apple docs cover this

## Knowing when things changed

Since data will be syncing from other clients, we need to know when objects in our UI have changed remotely.

- An NSFetchedResultsController will track it automatically, otherwise you need to listen for NSManagedObjectContextDidChange (does it really trigger for history?). You can also emit custom notifications during persistent history processing for application-specific changes.

## Create an NSFetchedResultsController
- Optionally wrap it behind a Store

## Implement NSFetchedResultsControllerDelegate
- Listen for changes
- Collect changes then flush
- Ignore user-initiation moves

## Enforce database constraints in NSPersistentHistoryProcessing

Since NSPersistentCloudKitContainer requires attributes and relationships to be optional and does not allow unique constraints, we need to enforce this ourselves.

## Implementing ordered relationships

NSPersistentCloudKitContainer does not allowed ordered relationships. To implement this, we need to add an index property to the child objects and use an NSSortPredicate to sort by this index while using an NSFetchedResultsController.

The alternative of storing index positions on the parent is not viable because it does not work with NSFetchedResultsController.
