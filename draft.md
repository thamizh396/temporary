


Logic for claiming the thread based failure or claim:

While assigning the partitions, each thread will have some certain batch start and end. So for each thread had to set a key value with the thread name plus some identifier and the value will have the start and end
Eg : thread_1_startend = start:101,end:200

Then once the process got ended then had to claim/delete the cache key as the entries has been processed.

Here comes the query, once on the same process end or on the next job, we should pick these missed out records and reprocess the data without any duplication.


Logic for start up:

We have to get the last processed id or the next processed from the cache key as below,
Last_processed_id: 100
Next_processed_id: 101

So if the key are not present in DB on first run or if cache gets down had to fetch these ids from the db of the max transaction ID of output or error table. Then consider this as the last processed id.

Then we have to
