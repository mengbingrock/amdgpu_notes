The sync obj info are passed in chunks.
```
	struct drm_amdgpu_cs_chunk chunks[2];

```
And it's value is assigned as below:

```
    chunks[0].chunk_id = AMDGPU_CHUNK_ID_IB;
	chunks[0].length_dw = sizeof(struct drm_amdgpu_cs_chunk_ib) / 4;
	chunks[0].chunk_data = (uint64_t)(uintptr_t)&chunk_data;
	chunk_data.ib_data._pad = 0;
	chunk_data.ib_data.va_start = ib_result_mc_address;
	chunk_data.ib_data.ib_bytes = 16 * 4;
	chunk_data.ib_data.ip_type = wait_or_signal ? AMDGPU_HW_IP_GFX :
		AMDGPU_HW_IP_DMA;
	chunk_data.ib_data.ip_instance = 0;
	chunk_data.ib_data.ring = 0;
	chunk_data.ib_data.flags = 0;

	chunks[1].chunk_id = wait_or_signal ?
		AMDGPU_CHUNK_ID_SYNCOBJ_TIMELINE_WAIT :
		AMDGPU_CHUNK_ID_SYNCOBJ_TIMELINE_SIGNAL;
	chunks[1].length_dw = sizeof(struct drm_amdgpu_cs_chunk_syncobj) / 4;
	chunks[1].chunk_data = (uint64_t)(uintptr_t)&syncobj_data;
	syncobj_data.handle = syncobj_handle;
	syncobj_data.point = point;
	syncobj_data.flags = DRM_SYNCOBJ_WAIT_FLAGS_WAIT_FOR_SUBMIT;
```
Then submit command and get the context_handle

```
r = amdgpu_cs_submit_raw(device_handle,
				 context_handle,
				 bo_list,
				 2,
				 chunks,
				 &seq_no);

```
Then allocate and initiate the fence to query its status:

```
    fence_status.context = context_handle;
	fence_status.ip_type = wait_or_signal ? AMDGPU_HW_IP_GFX:
		AMDGPU_HW_IP_DMA;
	fence_status.ip_instance = 0;
	fence_status.ring = 0;
	fence_status.fence = seq_no;
```
```
	r = amdgpu_cs_query_fence_status(&fence_status,
			AMDGPU_TIMEOUT_INFINITE,0, &expired);
	
```
To query the timeline payload.
```
	r = amdgpu_cs_syncobj_query(device_handle, &syncobj_handle,
				    &payload, 1);
```
the kernel has this funtion registered to respond this query request:
```
 drm_syncobj_query_ioctl


 
	for (i = 0; i < args->count_handles; i++) {
		struct dma_fence_chain *chain;
		struct dma_fence *fence;
		uint64_t point;

		fence = drm_syncobj_fence_get(syncobjs[i]);
		chain = to_dma_fence_chain(fence);
		if (chain) {
			struct dma_fence *iter, *last_signaled =
				dma_fence_get(fence);

			if (args->flags &
			    DRM_SYNCOBJ_QUERY_FLAGS_LAST_SUBMITTED) {
				point = fence->seqno;
			} else {
				dma_fence_chain_for_each(iter, fence) {
					if (iter->context != fence->context) {
						dma_fence_put(iter);
						/* It is most likely that timeline has
						* unorder points. */
						break;
					}
					dma_fence_put(last_signaled);
					last_signaled = dma_fence_get(iter);
				}
				point = dma_fence_is_signaled(last_signaled) ?
					last_signaled->seqno :
					to_dma_fence_chain(last_signaled)->prev_seqno;
			}
			dma_fence_put(last_signaled);
		} else {
			point = 0;
		}
		dma_fence_put(fence);
		ret = copy_to_user(&points[i], &point, sizeof(uint64_t));
 ```
This function will return points back to userspace, which is pay_load.

Then start a thread to set the wait_point to 16.
```
//signal on point 16
	sp3.syncobj_handle = syncobj_handle;
	sp3.point = 16;
	r = pthread_create(&c_thread, NULL, syncobj_signal, &sp3);
	
```
Then cpu wait on point 16

```

	//CPU wait on point 16
	wait_point = 16;
	timeout = 0;
	clock_gettime(CLOCK_MONOTONIC, &tp);
	timeout = tp.tv_sec * 1000000000ULL + tp.tv_nsec;
	timeout += 0x10000000000; //10s
	r = amdgpu_cs_syncobj_timeline_wait(device_handle, &syncobj_handle,
					    &wait_point, 1, timeout,
					    DRM_SYNCOBJ_WAIT_FLAGS_WAIT_ALL |
					    DRM_SYNCOBJ_WAIT_FLAGS_WAIT_FOR_SUBMIT,
					    NULL);


```



```
	// export point 16 and import to point 18
	r = amdgpu_cs_syncobj_export_sync_file2(device_handle, syncobj_handle,
						16,
						DRM_SYNCOBJ_WAIT_FLAGS_WAIT_FOR_SUBMIT,
						&sync_fd);
	CU_ASSERT_EQUAL(r, 0);
	r = amdgpu_cs_syncobj_import_sync_file2(device_handle, syncobj_handle,
						18, sync_fd);
	CU_ASSERT_EQUAL(r, 0);
	r = amdgpu_cs_syncobj_query(device_handle, &syncobj_handle,
				    &payload, 1);
	CU_ASSERT_EQUAL(r, 0);
	CU_ASSERT_EQUAL(payload, 18);

	// CPU signal on point 20
	signal_point = 20;
	r = amdgpu_cs_syncobj_timeline_signal(device_handle, &syncobj_handle,
					      &signal_point, 1);
	CU_ASSERT_EQUAL(r, 0);
	r = amdgpu_cs_syncobj_query(device_handle, &syncobj_handle,
				    &payload, 1);
	CU_ASSERT_EQUAL(r, 0);
	CU_ASSERT_EQUAL(payload, 20);


```




