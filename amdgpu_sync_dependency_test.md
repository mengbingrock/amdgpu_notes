static void amdgpu_sync_dependency_test(void)

The userspace struct to describe the request:
```

/**
 * Structure describing submission request
 *
 * \note We could have several IBs as packet. e.g. CE, CE, DE case for gfx
 *
 * \sa amdgpu_cs_submit()
*/
struct amdgpu_cs_request {
	/** Specify flags with additional information */
	uint64_t flags;

	/** Specify HW IP block type to which to send the IB. */
	unsigned ip_type;

	/** IP instance index if there are several IPs of the same type. */
	unsigned ip_instance;

	/**
	 * Specify ring index of the IP. We could have several rings
	 * in the same IP. E.g. 0 for SDMA0 and 1 for SDMA1.
	 */
	uint32_t ring;

	/**
	 * List handle with resources used by this request.
	 */
	amdgpu_bo_list_handle resources;

	/**
	 * Number of dependencies this Command submission needs to
	 * wait for before starting execution.
	 */
	uint32_t number_of_dependencies;

	/**
	 * Array of dependencies which need to be met before
	 * execution can start.
	 */
	struct amdgpu_cs_fence *dependencies;

	/** Number of IBs to submit in the field ibs. */
	uint32_t number_of_ibs;

	/**
	 * IBs to submit. Those IBs will be submit together as single entity
	 */
	struct amdgpu_cs_ib_info *ibs;

	/**
	 * The returned sequence number for the command submission 
	 */
	uint64_t seq_no;

	/**
	 * The fence information
	 */
	struct amdgpu_cs_fence_info fence_info;
};





/**
 * Structure describing submission request
 *
 * \note We could have several IBs as packet. e.g. CE, CE, DE case for gfx
 *
 * \sa amdgpu_cs_submit()
*/
struct amdgpu_cs_request {
	/** Specify flags with additional information */
	uint64_t flags;

	/** Specify HW IP block type to which to send the IB. */
	unsigned ip_type;

	/** IP instance index if there are several IPs of the same type. */
	unsigned ip_instance;

	/**
	 * Specify ring index of the IP. We could have several rings
	 * in the same IP. E.g. 0 for SDMA0 and 1 for SDMA1.
	 */
	uint32_t ring;

	/**
	 * List handle with resources used by this request.
	 */
	amdgpu_bo_list_handle resources;

	/**
	 * Number of dependencies this Command submission needs to
	 * wait for before starting execution.
	 */
	uint32_t number_of_dependencies;

	/**
	 * Array of dependencies which need to be met before
	 * execution can start.
	 */
	struct amdgpu_cs_fence *dependencies;

	/** Number of IBs to submit in the field ibs. */
	uint32_t number_of_ibs;

	/**
	 * IBs to submit. Those IBs will be submit together as single entity
	 */
	struct amdgpu_cs_ib_info *ibs;

	/**
	 * The returned sequence number for the command submission 
	 */
	uint64_t seq_no;

	/**
	 * The fence information
	 */
	struct amdgpu_cs_fence_info fence_info;
};


/**
 * Structure describing submission request
 *
 * \note We could have several IBs as packet. e.g. CE, CE, DE case for gfx
 *
 * \sa amdgpu_cs_submit()
*/
struct amdgpu_cs_request {
	/** Specify flags with additional information */
	uint64_t flags;

	/** Specify HW IP block type to which to send the IB. */
	unsigned ip_type;

	/** IP instance index if there are several IPs of the same type. */
	unsigned ip_instance;

	/**
	 * Specify ring index of the IP. We could have several rings
	 * in the same IP. E.g. 0 for SDMA0 and 1 for SDMA1.
	 */
	uint32_t ring;

	/**
	 * List handle with resources used by this request.
	 */
	amdgpu_bo_list_handle resources;

	/**
	 * Number of dependencies this Command submission needs to
	 * wait for before starting execution.
	 */
	uint32_t number_of_dependencies;

	/**
	 * Array of dependencies which need to be met before
	 * execution can start.
	 */
	struct amdgpu_cs_fence *dependencies;

	/** Number of IBs to submit in the field ibs. */
	uint32_t number_of_ibs;

	/**
	 * IBs to submit. Those IBs will be submit together as single entity
	 */
	struct amdgpu_cs_ib_info *ibs;

	/**
	 * The returned sequence number for the command submission 
	 */
	uint64_t seq_no;

	/**
	 * The fence information
	 */
	struct amdgpu_cs_fence_info fence_info;
};


```

If one job has dependencies on another, it will provide the depending fence seq_no when doing command submission.
```
ibs_request.number_of_dependencies = 1;

	ibs_request.dependencies = calloc(1, sizeof(*ibs_request.dependencies));
	ibs_request.dependencies[0].context = context_handle[1];
	ibs_request.dependencies[0].ip_instance = 0;
	ibs_request.dependencies[0].ring = 0;
	ibs_request.dependencies[0].fence = seq_no;


```
and then it submit job by calling:
```
drm_public int amdgpu_cs_submit_raw2(amdgpu_device_handle dev,
				     amdgpu_context_handle context,
				     uint32_t bo_list_handle,
				     int num_chunks,
				     struct drm_amdgpu_cs_chunk *chunks,
				     uint64_t *seq_no)

    r = amdgpu_cs_submit_raw2(dev, context, bo_list_handle, num_chunks,
				  chunks, &seq_no);
	
    chunk_array = alloca(sizeof(uint64_t) * num_chunks);
	for (i = 0; i < num_chunks; i++)
		chunk_array[i] = (uint64_t)(uintptr_t)&chunks[i];
	cs.in.chunks = (uint64_t)(uintptr_t)chunk_array;
	cs.in.ctx_id = context->id;
	cs.in.bo_list_handle = bo_list_handle;
	cs.in.num_chunks = num_chunks;
	r = drmCommandWriteRead(dev->fd, DRM_AMDGPU_CS,
				&cs, sizeof(cs));
	
```
The kernel will respond this request in ioctl:

```
int amdgpu_cs_ioctl(struct drm_device *dev, void *data, struct drm_file *filp)

```
The above func will parse the job from *data, which is union drm_amdgpu_cs. 
The goal of the parser is to fill the info to amdgpu_cs_parser.
```
struct amdgpu_cs_parser {
	struct amdgpu_device	*adev;
	struct drm_file		*filp;
	struct amdgpu_ctx	*ctx;

	/* chunks */
	unsigned		nchunks;
	struct amdgpu_cs_chunk	*chunks;

	/* scheduler job object */
	struct amdgpu_job	*job;
	struct drm_sched_entity	*entity;

	/* buffer objects */
	struct ww_acquire_ctx		ticket;
	struct amdgpu_bo_list		*bo_list;
	struct amdgpu_mn		*mn;
	struct amdgpu_bo_list_entry	vm_pd;
	struct list_head		validated;
	struct dma_fence		*fence;
	uint64_t			bytes_moved_threshold;
	uint64_t			bytes_moved_vis_threshold;
	uint64_t			bytes_moved;
	uint64_t			bytes_moved_vis;

	/* user fence */
	struct amdgpu_bo_list_entry	uf_entry;

	unsigned			num_post_deps;
	struct amdgpu_cs_post_dep	*post_deps;
};


```

This test we are interested in dependencies, it is contained in 
```
struct drm_amdgpu_cs_chunk {
	__u32		chunk_id;
	__u32		length_dw;
	__u64		chunk_data;
};
```
the chunk_id could take one of the following values:
```
#define AMDGPU_CHUNK_ID_IB		0x01
#define AMDGPU_CHUNK_ID_FENCE		0x02
#define AMDGPU_CHUNK_ID_DEPENDENCIES	0x03
#define AMDGPU_CHUNK_ID_SYNCOBJ_IN      0x04
#define AMDGPU_CHUNK_ID_SYNCOBJ_OUT     0x05
#define AMDGPU_CHUNK_ID_BO_HANDLES      0x06
#define AMDGPU_CHUNK_ID_SCHEDULED_DEPENDENCIES	0x07
#define AMDGPU_CHUNK_ID_SYNCOBJ_TIMELINE_WAIT    0x08
#define AMDGPU_CHUNK_ID_SYNCOBJ_TIMELINE_SIGNAL  0x09


```
the parser will parse the dependencies judging from the id:

```
switch (p->chunks[i].chunk_id) {
		case AMDGPU_CHUNK_ID_IB:
			++num_ibs;
			break;

		case AMDGPU_CHUNK_ID_FENCE:
			size = sizeof(struct drm_amdgpu_cs_chunk_fence);
			if (p->chunks[i].length_dw * sizeof(uint32_t) < size) {
				ret = -EINVAL;
				goto free_partial_kdata;
			}

			ret = amdgpu_cs_user_fence_chunk(p, p->chunks[i].kdata,
							 &uf_offset);
			if (ret)
				goto free_partial_kdata;

			break;

```
Then allocate the job entity.

```
	ret = amdgpu_job_alloc(p->adev, num_ibs, &p->job, vm);
```
the amdgpu_job id defined as:
```
struct amdgpu_job {
	struct drm_sched_job    base;
	struct amdgpu_vm	*vm;
	struct amdgpu_sync	sync;
	struct amdgpu_sync	sched_sync;
	struct amdgpu_ib	*ibs;
	struct dma_fence	*fence; /* the hw fence */
	uint32_t		preamble_status;
	uint32_t                preemption_status;
	uint32_t		num_ibs;
	bool                    vm_needs_flush;
	uint64_t		vm_pd_addr;
	unsigned		vmid;
	unsigned		pasid;
	uint32_t		gds_base, gds_size;
	uint32_t		gws_base, gws_size;
	uint32_t		oa_base, oa_size;
	uint32_t		vram_lost_counter;

	/* user fence handling */
	uint64_t		uf_addr;
	uint64_t		uf_sequence;
};


```
where uf_addr and uf_sequence is user fence address and sequence, respectively.


```
int amdgpu_job_alloc(struct amdgpu_device *adev, unsigned num_ibs,
		     struct amdgpu_job **job, struct amdgpu_vm *vm)
{
	size_t size = sizeof(struct amdgpu_job);

	if (num_ibs == 0)
		return -EINVAL;

	size += sizeof(struct amdgpu_ib) * num_ibs;

	*job = kzalloc(size, GFP_KERNEL);
	if (!*job)
		return -ENOMEM;

	/*
	 * Initialize the scheduler to at least some ring so that we always
	 * have a pointer to adev.
	 */
	(*job)->base.sched = &adev->rings[0]->sched;
	(*job)->vm = vm;
	(*job)->ibs = (void *)&(*job)[1];
	(*job)->num_ibs = num_ibs;

	amdgpu_sync_create(&(*job)->sync);
	amdgpu_sync_create(&(*job)->sched_sync);
	(*job)->vram_lost_counter = atomic_read(&adev->vram_lost_counter);
	(*job)->vm_pd_addr = AMDGPU_BO_INVALID_OFFSET;

	return 0;
}


```
The `amdgpu_sync_create` will initialize the hashtable.

```
/**
 * amdgpu_sync_create - zero init sync object
 *
 * @sync: sync object to initialize
 *
 * Just clear the sync object for now.
 */
void amdgpu_sync_create(struct amdgpu_sync *sync)
{
	hash_init(sync->fences);
	sync->last_vm_update = NULL;
}


```
After `	r = amdgpu_cs_parser_init(&parser, data)`, `r = amdgpu_cs_ib_fill(adev, &parser)` will fill in the ib information.

```
static int amdgpu_cs_ib_fill(struct amdgpu_device *adev,
			     struct amdgpu_cs_parser *parser)
{
        ...
		parser->entity = entity;
		ib->gpu_addr = chunk_ib->va_start;
		ib->length_dw = chunk_ib->ib_bytes / 4;
		ib->flags = chunk_ib->flags;
        ...

	return amdgpu_ctx_wait_prev_fence(parser->ctx, parser->entity);

```

The function `	return amdgpu_ctx_wait_prev_fence(parser->ctx, parser->entity)` will wait for previous job to finish?

After the `	return amdgpu_ctx_wait_prev_fence(parser->ctx, parser->entity)` is the `	r = amdgpu_cs_dependencies(adev, &parser)`



```
r = amdgpu_cs_dependencies(adev, &parser);
	if (r) {
		DRM_ERROR("Failed in the dependencies handling %d!\n", r);
		goto out;
	}

	r = amdgpu_cs_parser_bos(&parser, data);
	if (r) {
		if (r == -ENOMEM)
			DRM_ERROR("Not enough memory for command submission!\n");
		else if (r != -ERESTARTSYS && r != -EAGAIN)
			DRM_ERROR("Failed to process the buffer list %d!\n", r);
		goto out;
	}

	reserved_buffers = true;

	trace_amdgpu_cs_ibs(&parser);

	r = amdgpu_cs_vm_handling(&parser);
	if (r)
		goto out;

	r = amdgpu_cs_submit(&parser, cs);

```

