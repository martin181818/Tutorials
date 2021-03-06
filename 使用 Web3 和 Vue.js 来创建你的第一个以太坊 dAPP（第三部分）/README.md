## 使用 Web3 和 Vue.js 来创建你的第一个以太坊 dAPP（第三部分）

##### 上接第二部分


到目前为止，我们的应用程序能够从 metamask 获取并显示帐户数据。但是，在更改帐户时，如果不重新加载页面，则不会更新数据。这并不是最优的，我们希望能够确保响应式地更新数据。
我们的方法与简单地初始化 web3 实例略有不同。Metamask 还不支持 websockets，因此我们将不得不每隔一段时间就去轮询数据是否有修改。我们不希望在没有更改的情况下调度操作，因此只有在满足某个条件（特定更改）时，我们的操作才会与它们各自的有效负载一起被调度。
也许上述方法并不是诸多解决方案中的最优解，但是它在严格模式的约束下工作，所以还算不错。在 util 文件夹中创建一个名为 pollWeb3.js 的新文件。下面是我们要做的：

- 导入 web3，这样我们就不依赖于 Metamask 实例
- 导入我们的 store，这样我们就可以进行数据对比和分发操作
- 创建 web3 实例
- 设置一个间隔来检查地址是否发生了变化，如果没有，检查余额是否发生了变化
- 如果地址或余额有变化，我们将更新我们的 store。因为我们的 hello-metamask 组件具有一个 Computed 属性，这个改变是响应式的

![](https://i.imgur.com/2oQaQC4.png)
   

现在，一旦我们的 web3 实例被初始化，我们就要开始轮询更新。所以，打开 Store/index.js ，导入 pollWeb3.js 文件，并将其添加到我们的 regierWeb3Instance() 方法的底部，以便在状态更改后执行。


    import pollWeb3 from '../util/pollWeb3'

    registerWeb3Instance (state, payload) {
    console.log('registerWeb3instance Mutation being executed', payload)
    let result = payload
    let web3Copy = state.web3
    web3Copy.coinbase = result.coinbase
    web3Copy.networkId = result.networkId
    web3Copy.balance = parseInt(result.balance, 10)
    web3Copy.isInjected = result.injectedWeb3
    web3Copy.web3Instance = result.web3
    state.web3 = web3Copy
    pollWeb3()
    }

由于我们正在调度操作，所以需要将其添加到 store 中，并进行变异以提交更改。我们可以直接提交更改，但为了保持模式一致性，我们不这么做。我们将添加一些控制台日志，以便您可以在控制台中观看精彩的过程。在 actions 对象中添加：

    pollWeb3 ({commit}, payload) {
    console.log('pollWeb3 action being executed')
    commit('pollWeb3Instance', payload)
    }

搞定了！如果我们现在改变 Metamask 的地址，或者余额发生变化，我们将看到在我们的应用程序无需重新加载页面更新。当我们更改网络时，页面将重新加载，我们将重新注册一个新实例。但是，在生产中，我们希望显示一个警告，要求更改到部署协约的正确网络。

在下一节，我们将最终深入到我们的智能协议连接到我们的应用程序。与我们已经做过的相比，这实际上相当容易了。

### 实例化我们的协议

首先，我们将编写代码，然后部署协议并将 ABI 和 Address 插入到应用程序中。为了创建我们期待已久的 casino 组件，需要执行以下操作：

- 需要一个输入字段，以便用户可以输入下注金额
- 需要代表下注数字的按钮，当用户点击某个数字时，它将把输入的金额押在该数字上
- onClick 函数将调用 smart 协议上的 bet() 函数
- 显示一个加载旋转器，以显示事务正在进行中
- 交易完成后，我们会显示用户是否中奖以及中奖金额

但是，首先，我们需要我们的应用程序能够与我们的智能协议交互。我们将用已经做过的同样的方法来处理该问题。在 util 文件夹中创建一个名为 getContract.js 的新文件。

    import Web3 from ‘web3’
    import {address, ABI} from ‘./constants/casinoContract’

    let getContract = new Promise(function (resolve, reject) {
    let web3 = new Web3(window.web3.currentProvider)
    let casinoContract = web3.eth.contract(ABI)
    let casinoContractInstance = casinoContract.at(address)
    // casinoContractInstance = () => casinoContractInstance
    resolve(casinoContractInstance)
    })

    export default getContract

首先要注意的是，我们正在导入一个尚不存在的文件，稍后我们将在部署协议时修复该文件。

首先，我们通过将 ABI(我们将回到)传递到 web3.eth.Contact() 方法中，为稳固性协议创建一个协议对象。然后，我们可以在一地址上初始化该对象。在这个实例中，我们可以调用我们的方法和事件。

然而，如果没有 action 和变体，这将是不完整的。因此，在 casino-component.vue 的脚本标记中添加以下内容。


    export default {
    name: ‘casino’,
    mounted () {
    console.log(‘dispatching getContractInstance’)
    this.$store.dispatch(‘getContractInstance’)
    }
    }

现在 action 和变体在 store 中。首先导入 getContract.js 文件，我相信您现在已经知道如何做到这一点了。然后在我们创建的过程中，调用它：

    getContractInstance ({commit}) {
    getContract.then(result => {
    commit(‘registerContractInstance’, result)
    }).catch(e => console.log(e))
    }

把结果传给我们的变体：

    registerContractInstance (state, payload) {
    console.log(‘Casino contract instance: ‘, payload)
    state.contractInstance = () => payload
    }


这将把我们的协议实例存储在 store 中，以便我们在组件中使用。


与我们的协议交互

首先，我们将添加一个数据属性（在导出中）到我们的 casino 组件中，这样我们就可以拥有具有响应式属性的变量。这些值将是 winEvent、amount 和 Pending。

    data () {
    return {
    amount: null,
    pending: false,
    winEvent: null
    }
    }

我们将创建一个 onclick 函数来监听用户点击数字事件。这将触发协议上的 bet() 函数，显示微调器，当它接收到事件时，隐藏微调器并显示事件参数。在 data 属性下，添加一个名为 methods 的属性，该属性接收一个对象，我们将在其中放置我们的函数。

    methods: {
    clickNumber (event) {
      console.log(event.target.innerHTML, this.amount)
      this.winEvent = null
      this.pending = true
      this.$store.state.contractInstance().bet(event.target.innerHTML, {
        gas: 300000,
        value: this.$store.state.web3.web3Instance().toWei(this.amount, 'ether'),
        from: this.$store.state.web3.coinbase
      }, (err, result) => {
        if (err) {
          console.log(err)
          this.pending = false
        } else {
          let Won = this.$store.state.contractInstance().Won()
          Won.watch((err, result) => {
            if (err) {
              console.log('could not get event Won()')
            } else {
              this.winEvent = result.args
              this.pending = false
            }
          })
        }
      })
    }
    }


bet() 函数的第一个参数是在协议中定义的参数 u Number.Event.Target.innerHTML ，接下来，引用我们将在列表标记中创建的数字。然后是一个定义事务参数的对象，这是我们输入用户下注金额的地方。第三个参数是回调函数。完成后，我们将监听这一事件

现在，我们将为组件创建 html 和 CSS。只是复制粘贴它，我认为它已经很浅显了。在此之后，我们将部署协议，并获得 ABI 和 Address。

    <template>
    <div class=”casino”>
     <h1>Welcome to the Casino</h1>
    <h4>Please pick a number between 1 and 10</h4>
    Amount to bet: <input v-model=”amount” placeholder=”0 Ether”>
    <ul>
     <li v-on:click=”clickNumber”>1</li>
     <li v-on:click=”clickNumber”>2</li>
     <li v-on:click=”clickNumber”>3</li>
     <li v-on:click=”clickNumber”>4</li>
     <li v-on:click=”clickNumber”>5</li>
     <li v-on:click=”clickNumber”>6</li>
     <li v-on:click=”clickNumber”>7</li>
     <li v-on:click=”clickNumber”>8</li>
     <li v-on:click=”clickNumber”>9</li>
     <li v-on:click=”clickNumber”>10</li>
    </ul>
    <img v-if=”pending” id=”loader” src=”https://loading.io/spinners/double-ring/lg.double-ring-spinner.gif”>
    <div class=”event” v-if=”winEvent”>
    Won: {{ winEvent._status }}
    Amount: {{ winEvent._amount }} Wei
    </div>
    </div>
    </template>

    <style scoped>
    .casino {
    margin-top: 50px;
    text-align:center;
    }
    #loader {
    width:150px;
    }
    ul {
    margin: 25px;
    list-style-type: none;
    display: grid;
    grid-template-columns: repeat(5, 1fr);
    grid-column-gap:25px;
    grid-row-gap:25px;
    }
    li{
     padding: 20px;
     margin-right: 5px;
     border-radius: 50%;
     cursor: pointer;
     background-color:#fff;
     border: -2px solid #bf0d9b;
     color: #bf0d9b;
     box-shadow:3px 5px #bf0d9b;
    }
    li:hover{
    background-color:#bf0d9b;
    color:white;
    box-shadow:0px 0px #bf0d9b;
    }
    li:active{
     opacity: 0.7;
    }
     *{
     color: #444444;
    }
    </style>



### Ropsten 网络和 Metamask（面向第一次用户）

如果您不熟悉 metamask 或以太坊网络，请不要担心。

- 打开浏览器和 metamask 插件。接受使用条款并创建密码。
- 将种子短语存放在安全的地方(这是为了在丢失钱包时将其恢复原状)。
- 点击「以太坊主网」并切换到 Ropsten 测试网。
- 单击「购买」，然后单击「Ropsten Testnet Fucet」。在这里我们可以得到一些免费的测试-以太坊。
- 在 faucet 网站上，点击「从 faucet 请求 1 ether」几次。

当所有的事情都熟悉了并做完之后，您的 Metamask 应该如下所示：

![](https://i.imgur.com/7GiL0d0.png)

### 部署和连接

再打开 remix，我们的协议应该还在。如果不是，请转到此要点并复制粘贴。在 ReMix 的 rop 右边，确保我们的环境被设置为「InsistedWeb 3(Ropsten)」，并且选择了我们的地址。

部署与第1部分中的部署相同。我们在 Value 字段中输入几个参数来预装协议，输入构造函数参数，然后单击 Create。这一次，metamask 将提示接受/拒绝事务（约定部署）。单击「接受」并等待事务完成。

当 TX 完成后点击它，这将带你到那个 TX 的萎缩块链浏览器。我们可以在「to」字段下找到协议的地址。你的协议虽然不同，但看起来很相似。

![](https://i.imgur.com/Yw4QGA3.png)

我们的协议地址在「to」字段中。

这就给了我们地址，现在是 ABI。回到 remix 并切换到「编译」选项卡（右上角）。在协议名称旁边，我们将看到一个名为「Details」的按钮，单击它。第四个领域是我们的 ABI。

不错，现在我们只需要创建前一节还不存在的一个文件。因此，在 util/constents 文件夹中创建一个名为 casinoContract.js 的新文件。创建两个变量，粘贴必要的内容并导出变量，这样我们从上面导入的内容就可以访问它们。

    const address = ‘0x…………..’
    const ABI = […]
    export {address, ABI}

现在，我们可以通过在终端中运行 npm start ，并在浏览器中运行 localhost：8080 来测试我们的应用程序。输入金额并单击一个数字。Metamask 将提示您接受事务，旋转器将启动。在 30 秒到 1 分钟之后，我们得到第一次确认，因此也得到了事件的确认。我们的余额发生了变化，所以 pollweb 3 触发它的 action 来更新余额：

![](https://i.imgur.com/343AFSa.png)

最终结果(左)和生命周期(右)。

如果你能在这个系列中走到这一步，我会为您鼓掌。我不是一个专业的作家，所以有时阅读起来并不容易。我们的应用程序在主干网上已经设置好了，我们只需要让它更漂亮一些，更友好一些。我们将在下一节中这样做，尽管这是可选的。


### 关注需要它的部分

我们很快就会讲完的。它将只是一些 html、css 和 vue 条件语句，带有 v-if/v-Else。

**在 App.vue **中，将容器类添加到我们的 div 元素中，在 CSS 中定义该类：

    .container {
    padding-right: 15px;
    padding-left: 15px;
    margin-right: auto;
    margin-left: auto;
    }
    @media (min-width: 768px) {
     .container {
     width: 750px;
     }
    }

**在 main.js 中，**导入我们已经安装的 font-awesome 的库（我知道，这不是我们需要的两个图标的最佳方式）：

    import ‘font-awesome/css/font-awesome.css’


在 Hello-metanask.vue 中，我们将做一些更改。我们将在我们的 Computed 属性中使用 mapState 助手，而不是当前函数。我们还将使用 v-if 检查 isInjected ，并在此基础上显示不同的 HTML。最后的组件如下所示：

    <template>
    <div class='metamask-info'>
    <p v-if="isInjected" id="has-metamask"><i aria-hidden="true" class="fa fa-check"></i> Metamask installed</p>
    <p v-else id="no-metamask"><i aria-hidden="true" class="fa fa-times"></i> Metamask not found</p>
    <p>Network: {{ network }}</p>
    <p>Account: {{ coinbase }}</p>
    <p>Balance: {{ balance }} Wei </p>
     </div>
    </template>

    <script>
    import {NETWORKS} from '../util/constants/networks'
    import {mapState} from 'vuex'
    export default {
    name: 'hello-metamask',
    computed: mapState({
    isInjected: state => state.web3.isInjected,
    network: state => NETWORKS[state.web3.networkId],
    coinbase: state => state.web3.coinbase,
    balance: state => state.web3.balance
    })
    }
    </script>

    <style scoped>
    #has-metamask {
    color: green;
    }
    #no-metamask {
     color:red;
    }</style>

我们将执行相同的 v-if/v-else 方法来设计我们的事件，该事件将在赌场内部返回 -Component.vue：

    <div class=”event” v-if=”winEvent”>
     <p v-if=”winEvent._status” id=”has-won”><i aria-hidden=”true” class=”fa fa-check”></i> Congragulations, you have won {{winEvent._amount}} wei</p>
    <p v-else id=”has-lost”><i aria-hidden=”true” class=”fa fa-check”></i> Sorry you lost, please try again.</p>
    </div>

    #has-won {
    color: green;
    }
    #has-lost {
    color:red;
    }

最后，在我们的 clickNumber() 函数中，在 this.winEvent=Result.args ：下面添加一行：

    this.winEvent._amount = parseInt(result.args._amount, 10)