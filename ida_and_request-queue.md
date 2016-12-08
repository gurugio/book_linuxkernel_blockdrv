# ida를 이용한 request-queue의 id 생성


blk_alloc_queue_node함수를 보면 request-queue의 id를 지정하는 코드가 있습니다.
```
struct request_queue *blk_alloc_queue_node(gfp_t gfp_mask, int node_id)
{
    struct request_queue *q;
	int err;

	q = kmem_cache_alloc_node(blk_requestq_cachep,
				gfp_mask | __GFP_ZERO, node_id);
	if (!q)
		return NULL;

	q->id = ida_simple_get(&blk_queue_ida, 0, 0, gfp_mask);
	if (q->id < 0)
		goto fail_q;
```
ida_simple_get이라는 함수인데, 여러개의 request-queue가 있을때 각각의 고유한 id를 만들어주는 함수입니다. 이번장은 ida가 어떻게 구현되는지를 알아보겠습니다.

id를 만든다는 것은 간단한 정수 카운터로 만드는 방법도 있겠지만, 정수 카운터가 변수값의 한계까지 증가해서 다시 0이 되버린다면 어떻게 될까요? 또 정수 카운터의 값이 5인데 0번 id가 해지되었다면, 다음 id를 할당할때 5를 할당해야할까요 아니면 0을 할당해야할까요. 0을 할당해야된다면 정수 카운터만으로 0을 할당할 방법이 있을까요?

결국 id 번호를 만드는 것조차 커널처럼 매우 오랫동안 매우 많은 자원을 다루는 대규모 소프트웨어에게는 간단한 문제가 아니라는 것을 알 수 있습니다. 그래서 idr이나 ida같은 알고리즘들이 생겨났습니다. 

ida는 idr을 기반으로 만들어졌으니 먼저 idr 코드를 읽어보고 ida를 읽어보겠습니다.

##idr

ida는 idr을 기반으로 구현되었기 때문에 먼저 idr을 알아보겠습니다.

idr은 원래 포인터의 고유한 id를 만들기위해 만들어졌습니다. 사용자의 요청에 따라 커널이 객체를 만들었을때, 사용자에게 커널 객체의 포인터를 넘겨줄 수는 없습니다. 넘겨줘도 사용자는 포인터의 데이터를 읽을 수 없을뿐 아니라, 보안상으로도 옳지 않으니까요. 그래서 사용자에게는 id를 넘겨주고, 사용자가 해당 id를 사용할때마다 커널은 id를 가지고 어떤 커널 객체인지 찾아서 사용하도록 하는 인터페이스가 필요했습니다. 그래서 idr이라는게 만들어졌습니다.

결국 하나의 포인터와 정수 id를 매핑해주는게 idr의 역할입니다.

참고자료
* http://studyfoss.egloos.com/5187192

###struct idr

idr의 기본 자료구조부터 보겠습니다.
```
#define IDR_BITS 8
#define IDR_SIZE (1 << IDR_BITS)
#define IDR_MASK ((1 << IDR_BITS)-1)

struct idr_layer {
    int			prefix;	/* the ID prefix of this idr_layer */
	int			layer;	/* distance from leaf */
	struct idr_layer __rcu	*ary[1<<IDR_BITS];
	int			count;	/* When zero, we can release it */
	union {
		/* A zero bit means "space here" */
		DECLARE_BITMAP(bitmap, IDR_SIZE);
		struct rcu_head		rcu_head;
	};
};

struct idr {
	struct idr_layer __rcu	*hint;	/* the last layer allocated from */
	struct idr_layer __rcu	*top;
	int			layers;	/* only valid w/o concurrent changes */
	int			cur;	/* current pos for cyclic allocation */
	spinlock_t		lock;
	int			id_free_cnt;
	struct idr_layer	*id_free;
};
```
idr_layer는 트리 구조를 가집니다. ary 배열에 하위 트리의 주소를 가지거나 아니면, id를 부여할 포인터 값을 가지게됩니다. ary배열의 크기가 256개입니다. 즉 하나의 트리 노드가 256개의 포인터를 가질 수 있다는 말이됩니다. 그러므로 트리의 높이가 2인 트리라면 256*256, 약 64000개의 포인터를 저장할 수 있겠네요. 하나의 트리 노드에 몇개의 포인터가 저장되어있는지는 count 필드에 저장됩니다. 그리고 bitmap을 둬서 매번 ary 배열 전체를 다 조사하지 않고 ary배열에서 빈 곳을 찾을 수 있게 했습니다.

idr 구조체는 idr을 대표하는 구조체입니다. top 필드가 idr_layer 트리의 포인터입니다. 다른 필드는 코드를 분석하면서 알아보겠습니다.

###idr_alloc

idr_alloc은 idr_get_empty_slot함수와 idr_fill_slot함수로 구현됩니다.

idr_get_empty_slot로 id값을 찾습니다. 그리고 idr_fill_slot함수로 해당 id에 매핑하려는 포인터 주소 ptr를 매핑합니다.

###idr_get_empty_slot

함수의 인자부터 보겠습니다.
* idp: idr 구조체의 객체, DEFINE_IDR매크로로 선언하거나 idr_init으로 생성합니다.
 * id값이 시스템 전체적으로 고유한 값이 아니라 하나의 idr 객체를 사용하는 모듈에서만 고유한 것입니다.
* starting_id: 이 값보다 크거나 같은 id를 찾습니다.
* pa: idr_layer 트리를 순회할때 임시로 사용할 포인터 배열입니다.
* gfp_mask: idr_layer 노드를 할당할때 사용할 플래그
* layer_idr: 사용하지 않음

함수가 시작되면 먼저 idr_max함수를 호출해서 현재 트리에서 할당하능한 id의 갯수를 확인합니다. 만약 starting_id가 현재 트리에서 할당가능한 id라면 sub_alloc을 호출합니다.

그리고 starting_id가 현재 트리에서 할당가능하지 않다면 트리의 높이를 늘려야합니다. idr_layer_alloc함수 새로운 트리 노드를 할당하는 함수입니다. 새로운 트리 노드를 할당했으면 다음과 같이 새로운 트리 노드를 초기화합니다.
* ary[0]=p: 배열의 첫번째 항목에 기존 트리의 루트 노드를 저장합니다.
* count=1: 새로운 노드의 자식은 한개뿐입니다.
* layer=layers-1: layers값은 idr 구조체의 layers필드에 들어갈 값입니다. 루트 노드의 layer값은 항상 layers값보다 1작습니다. 어쨌든 새로운 높이로 설정합니다.
* __set_bit(0, new->bitmap): 비트맵에서 0번 비트를 1로 초기화합니다. 새로 생성한 노드이므로 다른쓰레드에서 사용될수없습니다. 그래서 락이 필요없는 __set_bit를 사용합니다.
* p=new: p는 idp->top에 쓸 값입니다. 즉 idp의 루트 노드 포인터를 새로운 노드로 바꿉니다.
 * idp->top에 새로운 값 p를 쓸때는 rcu_assign_pointer를 사용합니다. idr 객체에서 top필드는 rcu로 보호되는 값입니다. 따라서 값을 바꿀때는 rcu에서 제공하는 매크로들을 사용해야합니다.

이제 sub_alloc을 호출할 준비가 끝났습니다. sub_alloc의 인자는 idr_get_empty_slot과 동일합니다. 단지 starting_id 인자가 포인터로 바꼈습니다. 여기에서는 트리의 깊이를 늘리지않고, 필요에따라 노드만 늘립니다. 새로운 노드를 할당할때는 마찬가지로 idr_layer_alloc을 사용합니다. 하지만 새로운 노드를 부모 노드의 ary배열에 저장합니다. 그리고 부모 노드의 count를 증가시킵니다.

예를 들어 트리의 높이가 1이면 pa[1]에 트리의 루트 노드 포인터가 저장됩니다.  그리고 루트노드의 bitmap에서 0인 비트를 찾습니다. 그리고 ary배열에서 자식 노드의 포인터를 읽습니다. pa[0]에 자식노드의 포인터가 저장되고, 자식 노드의 bitmap 배열에서 0인 비트를 찾습니다. 그러면 id값이 찾아진 것입니다.

이제 pa 배열과 id 값이 idr_fill_slot함수로 전달될 것입니다.

###idr_fill_slot

pa배열의 0번 항목이 바로 id가 할당된 트리의 노드입니다. 따라서 pa[0]->ary[id] 값에 매핑될 포인터를 저장합니다. 이때도 ary 배열도 rcu로 보호되는 자원이므로 rcu_assign_pointer를 사용합니다.

idr_mark_full은 pa[0]의 bitmap에서 id에 해당하는 비트를 1로 셋팅합니다. 그런데 만약 pa[0]의 bitmap이 모두 1이라면 더이상 이 노드는 사용할 수 없으므로, 부모 노드로 올라가면서 비트맵을 1로 셋팅해야합니다. 이럴때 사용하기위해 pa포인터 배열을 이용한 것입니다.

##ida
ida는 ida_get_new_above함수가 핵심입니다.

ida_get_new_above함수는 idr_alloc과 같이 idr_get_empty_slot을 호출해서 pa 배열과 idr_id 값을 얻습니다. idr_alloc은 idr_fill_slot으로 포인터를 트리 노드에 저장했지만 ida는 저장할 포인터가 있는게 아니므로 idr_fill_slot을 호출하지 않습니다.

ida는 idr의 ary 배열을 idr_layer의 포인터로 쓰지 않습니다. 왜냐면 저장할 포인터가 없기 때문입니다. 대신에 비트맵의 포인터로 사용합니다. ida->free_bitmap에 미리 적당한 크기의 비트맵을 할당해놨다가 pa->ary 배열에 비트맵을 포인터를 저장해놓습니다.한 노드에 저장될 비트의 수는 IDA_BITMAP_BITS입니다. 그러므로 최종 id는 "idr_id(노드의 인덱스) * IDA_BITMAPBITS + 비트인덱스"가 됩니다.

만약 모든 비트맵이 1이면 idr_id를 증가합니다. 즉 다음 노드를 가져오는 것입니다. 다음 노드에 비트맵 자체가 아예 없으면 -EAGAIN을 반환합니다.





