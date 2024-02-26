

## 4类代码说明
`DILI.h` 中定义了dili本体的接口，其使用了`src/competitor/dili/src/src/global/global_typedef`中定义的`keyType`为key的类型，为`long`。
`GRE`中默认定义key类型为`unsigned long`。

第一类：原装dili，即github上dili代码部分未做修改，在GRE接口中为匹配dili中为long的`keyType`，benchmark的key也设置为long类型。

第二类：原装dili+keyType，github上dili代码部分将两处的`keyType`修改为unsigned long，GRE接口处使用GRE自带的unsigned long。
```
/home/gejiake/GRE_DILIoriginal/src/competitor/dili/src/src/global/global_typedef.h:19
typedef long keyType;

/home/gejiake/GRE_DILIoriginal/src/competitor/dili/src/src/butree/interval.cpp:6
const long *interval::data = NULL;
```

第三类：dili_debug: 记录函数周边数据
在第一类的基础上对diliNode.h等文件修改，主要涉及对溢出，精度等问题的尝试解决。

第四类：dili_debug+keyType
结合第二类和第三类，即debug的基础上对dili的`keyType`修改。


以下均为bulkload阶段出错
## 第一类
covid下报错
两种错误
### 1 put_three_keys
```
触发下面函数中的assert
assert(pos0 < pos1 && pos1 < pos2);
```
首先：keytype是long类型的，会有一些错误
a很小
有时候会调用put_three_keys，里面a很小会导致3个键被映射到一个位置，触发assert。
```cpp
in src/dili/diliNode.h
 inline void put_three_keys(const keyType *_keys, const recordPtr *_ptrs) {
        keyType k0 = _keys[0];
        keyType k1 = _keys[1];
        keyType k2 = _keys[2];
        double offset = fanout / 3.0;
        keyType s = MIN_KEY(k1 - k0, k2 - k1);
        b = offset / s;
        a = 0.5 + offset - b * k1;
        int pos0 = LR_PRED(a, b, k0, fanout);
        int pos1 = LR_PRED(a, b, k1, fanout);
        int pos2 = LR_PRED(a, b, k2, fanout);

        pe_data[pos0].assign(k0, _ptrs[0]);
        pe_data[pos1].assign(k1, _ptrs[1]);
        pe_data[pos2].assign(k2, _ptrs[2]);
        assert(pos0 < pos1 && pos1 < pos2);
        total_n_travs = 3;
        avg_n_travs_since_last_dist = 1;
}
```
定位到的数据
```
k0: 1344799332403441664
k1: 1344799332403441666
k2: 1344799333191999489
a: -1.3448e+18    b: 1
fanout: 6
pos0: 0
pos1: 0
pos2: 5
```


```
a 很小
LR_PRED 就是a+bx再截断
导致pos0 = pos1 ，触发assert
```

### 2 distribute_keys
```
void distribute_data(const keyType *keys, const recordPtr *ptrs, bool print=false) {
        assert(num_nonempty > 3);
        total_n_travs = 0;
        linearReg_w_expanding(keys, a, b, num_nonempty, fanout, false);
        int last_k_id = 0;
        keyType last_key = keys[0];
        int pos = -1;
        int last_pos = LR_PRED(a, b, last_key, fanout);
        keyType final_key = keys[num_nonempty - 1];
        if (b < 0 || last_pos == LR_PRED(a, b, final_key, fanout)) {
            linearReg_w_expanding(keys, a, b, num_nonempty, fanout, true);
            last_pos = LR_PRED(a, b, last_key, fanout);
            int final_pos = LR_PRED(a, b, final_key, fanout);
            
            assert(last_pos != final_pos);
            
        }
        assert(b >= 0);
```
原dili的策略是先求斜率b。如果斜率b小于0，或斜率b很小以至于所有键映射到一个地方，则用另一种方法(`linearReg_w_expanding(keys, a, b, num_nonempty, fanout, true);`)求b，然后设置assert检查新的b会不会还是太小。
由于有时a为一个绝对值很大的负数，b通过两个函数算出来都很小，第一个key和最后一个key总是映射到一起，从而触发assert。在第三类代码中，我通过用中间点来定位线段保证中间的key映射到中间，第一个key和最后一个key分居两侧来避免这个assert。
下面是出问题的数据。num_nonempty=4，dili使用linearReg_w_expanding计算a，b。
```
long long k0 = 1344795478643400706;
long long k1 = 1344795479247351808;
long long k2 = 1344795480694222850;
long long k3 = 1344795482397110272;
不行

long long k0 =1261237334017; // 0
long long k1 =1262248132608; // 1
long long k2 =1265028976640; // 2 left
long long k3 =1265028976654; // 3 
可以
```

根据以上两个问题修改代码形成第三类，具体为使用中间点来定位线段，同时尽量从数学上消去中间值提高精确度。
## 第二类
osm,fbu和fb数据集均有以下报错
```
microbench: /home/gejiake/GRE_DILIoriginal/src/competitor/dili/src/src/butree/buInterval.cpp:13: virtual void buInterval::init_merge_info(): Assertion `lbd < ubd' failed.

/home/gejiake/GRE_DILIoriginal/src/competitor/dili/src/src/butree/interval_utils.cpp：73
in partition():93
```
定位到
```
/home/gejiake/GRE_DILIoriginal/src/competitor/dili/src/src/butree/interval_utils.cpp:1001

for (long i = 2; i < N; i += 2) {
        keyType x = X[i];
        keyType x2 = X[i + 1] + 1;
        int fan = 2;
        if (i + 3 >= N) {
            fan = N - i;
            x2 = X[i + fan - 1] + 1;
        }
        interval *i_ptr = intervalInstance::newInstance(interval_type);
        i_ptr->fanout = fan;
        if (probs) {
            lSib->prob_sum = probs[i] + probs[i + 1];
            if (fan > 2) {
                lSib->prob_sum += probs[i + 1];
            }
        } else {
            lSib->prob_sum = single_prob * fan;
        }
        i_ptr->start_idx = i;
        i_ptr->end_idx = i + fan;
        
        i_ptr->lbd = x;
        last_ubd = i_ptr->ubd = x2;
        
        lSib->rSib = i_ptr;
        i_ptr->lSib = lSib;
        
        if(i_ptr->lbd >= i_ptr->ubd){
	        cout << "func get_complete_partition_borders" << endl;
            cout << "N in interval_utils.cpp" << N << endl;
            cout << 'i '<< i;
            cout << "x" << x << "   x2" << x2 << endl;
            cout << "X[i]" << X[i] << " X[i+1]" << X[i+1] << " X[N-1]"<< X[N-1] << endl;
        }
        
        i_ptr->init_merge_info();
```

调试出相关变量值，发现
```
x : 9223372036837191680
x2: 9223372036854775809
    18446744073709551615 max unsigned long 
    9223372036854775807  max long    
```

原始定义里面为long，可能是这里出了问题。
```
in /home/gejiake/GRE_DILIoriginal/src/competitor/dili/src/src/butree/interval.h

struct interval{
    static const keyType *data;
    static const double *probs;
    bool valid;
    int fanout;
    
    long lbd;
    long ubd; // lbd < x <= ubd
    
//    double rj;
//    double tj;
```
测试
```
#include<iostream>
#include<stdio.h>

int main(){
    using namespace std;
    long long l = 9223372036837191680;
    long long r = 9223372036854775809;
    if( l < r ) cout << "hello" << endl;
    cout << sizeof(long) << endl;
}
```
不会输出hello，说明在机器上，x >= x2。
## 第三类
osm下
```cpp
in /home/gejiake/GRE_DILIdebug/src/competitor/dili/src/src/dili/diliNode.h/void distribute_data(const keyType *keys, const recordPtr *ptrs, bool print=false)

pos = std::min<long long>(std::max<long long>(( node_mid_range + b * key - b* node_mid_key), 0), fanout - 1);
```
mid_range+b*(k-mid_key)
当k是负数，mid_key也是负数，会溢出，成为整数，从而导致第一个key的位置在整个段的中间，触发assert
可能由于将数据以long而非unsigned long读入，故执行第四类实验。

## 第四类
osm下会出现和第二类一样的问题，说明尚未涉及到第三类中遇到的问题时便会有第二类的问题。而第二类的问题是因为dili硬编码lbd的类型为long，如修改牵涉颇多且无道理。


## others
todo:
将dili_debug中keytype修改，在osm下测试，看是否通过（第四类 no）
todo：
解析原装dili在covid的错误。(done)
todo：
dili_unsigned osm解析错误（why ）
todo：
找回第一类的第一个错误(doing)