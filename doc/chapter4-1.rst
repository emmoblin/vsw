データプレーンのアーキテクチャ
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

データプレーンにおいてパケット処理をする機能は、モジュールとして実装される。
モジュールは、モジュールを束ねる vswitch インスタンスに紐付き、
同じ vswitch インスタンスに所属するモジュール間でパケットの授受が可能になる。
更に複数の vswitch インスタンスが、dispatcher インスタンスによって束ねることが可能になっている。[#note1]_ 
データプレーンの起動・停止は、dispatcher インスタンスへの指示となる。

.. [#note1] 将来的にインスタンスとしての dispatcher は無くし、vswitch インスタンスに統合する。

モジュール間は DPDK Ring によって相互につながる。
DPDK Ring 上は Lagopus で規定するメタデータを含む DPDK Mbuf が流れる。
DPDK Mbuf は、データプレーン部として確保する共通の DPDK Mempool より確保されるものとする。

モジュール
^^^^^^^^^^

パケット処理を行うモジュールはクラスとして定義されており、
必要に応じてインスタンス化された上で、パケットを処理するモジュールとしてデータプレーンに組み込まれる。

モジュール・クラスは ``vswtich.RegisterClass()`` 関数で登録する。
登録には以下の情報を渡す:

* 名前
* ``vswitch.Moduler`` インタフェースに準拠したインスタンス
* DPDK のスレーブ・コアの要否
* VIF であるか否か

``vswitch.Moduler`` インタフェースは以下のように定義される::

    type Moduler interface {
        New() interface{}
        Control(c string, v interface{}) interface{}
    }

``New()``
	``vswitch.ModulerInstancer`` インタフェースに準拠したモジュール・インスタンスの生成

``Control()``
	モジュール・クラスの制御。実装依存。

モジュール・インスタンス
^^^^^^^^^^^^^^^^^^^^^^^^

モジュールがインスタンス化されると、``vswitch.ModuleInstance`` 型の構造体を介して以下の情報が与えられる：

* 名前
* 入力用 DPDK Ring
* 出力先 DPDK Ring 判定ルール
* 使用する DPDK スレーブ・コア (Optional)
* VIF 設定情報 (Optional)

これらの情報へのポインタは、インスタンスの初期化時に渡される。
以下はモジュール・インスンスとして、実装する必要があるインタフェースである::

    type ModuleInstancer interface {
        Init(m *ModuleInstance)
        RingParam() *RingParam
        Start() int
        Stop()
        Wait()
        Control(c string, v interface{}) interface{}
    }

``Init()``
	モジュール・インスタンスの初期化。コア部で準備した ``ModuleInstance`` へのポインタが渡される。

``RignParam()``
	モジュール・インスタンスが必要とする入力用 DPDK Ring のパラメータを返す関数。``nil`` を返すとデフォルトの値で DPDK Ring は精製される。

``Start()``
	モジュール・インスタンスの開始。起動成功後は速やかに抜けること。成功した場合は 0、失敗した場合は -1 を返すこと。
	なお、モジュール・インスタンスが動作するために必要な情報は、本関数が呼ばれるまで確定しない可能性が高い。

``Stop()``
	モジュール・インスタンスの停止。停止指示成功後は速やかに抜けること。停止完了まで待たない。

``Wait()``
	モジュール・インスタンスの停止確認。停止確認ができるまでブロックすること。

``Control()``
	モジュール・インスタンスの制御。実装依存。


パケットの授受
^^^^^^^^^^^^^^

各モジュール・インスタンスは、自モジュール・インスタンス宛の DPDK Mbuf を入力用 DPDK Ring から dequeue し、
適切な処理を行った上で、次のモジュールの入力用 Ring に enqueue する。

出力先の DPDK Ring は、各モジュールが規定するルールに沿って決定するが、
その判定のために用いられる情報は「出力先 DPDK Ring 判定ルール」として渡される。
各判定ルールは、条件、条件パラメータ、条件にマッチした場合に送る先となる DPDK ring から構成される。
各モジュールがどの条件マッチに対応するかは、モジュールの実装依存である。[#note2]_

.. [#note2] 現状はモジュールがどの条件に対応しているか取得する方法がないが、将来的には用意する予定である。
