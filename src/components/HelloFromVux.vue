<template>
  <div>
    <ButtonTab>
      <button-tab-item selected @on-item-click="cuTab='1'">我的持仓</button-tab-item>
      <button-tab-item @on-item-click="cuTab='2'">交易记录</button-tab-item>
    </ButtonTab>
    <div class="tab-swiper vux-center" v-if="cuTab=='1'">
      <load-more :show-loading="true" background-color="#fbf9fe">持仓</load-more>
      <x-table>
        <thead>
          <tr>
            <th>名称</th>
            <th>代码</th>
            <th>现价</th>
            <th>市值</th>
            <th>数量</th>
            <th>成本</th>
            <th>盈亏</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="item in interest" :key="{item}">
            <td>{{item.name}}</td>
            <td>{{item.code}}</td>
            <td>{{item.price}}</td>
            <td>{{item.market}}</td>
            <td>{{item.num}}</td>
            <td>{{item.buyPrice}}</td>
            <td>{{item.profitLoss}}</td>
          </tr>
        </tbody>
      </x-table>

      <load-more :show-loading="true" background-color="#fbf9fe">清仓</load-more>
      <x-table>
        <thead>
          <tr>
            <th>名称</th>
            <th>代码</th>
            <th>现价</th>
            <th>盈亏</th>
            <th>清仓时间</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="item in clearance" :key="{item}">
            <td>{{item.name}}</td>
            <td>{{item.code}}</td>
            <td>{{item.price}}</td>
            <td>{{item.profitLoss}}</td>
            <td>{{item.ctime}}</td>
          </tr>
        </tbody>
      </x-table>
    </div>
    <div class="tab-swiper vux-center" v-if="cuTab=='2'">
      <flexbox class="flex-demo">
        <flexbox-item>
          <x-button type="primary" link="/demo">添加</x-button>
        </flexbox-item>
        <flexbox-item>
          <div class="fileBut weui-btn weui-btn_primary">
            <div class="fileBut2">导入</div>
            <input
              type="file"
              class="file_input"
              accept=".csv"
              ref="file"
              @change="tirggerFile($event)"
            />
          </div>
        </flexbox-item>
        <flexbox-item>
          <x-button type="primary" @click.native="exCsv">导出</x-button>
        </flexbox-item>
      </flexbox>

      <load-more :show-loading="true" background-color="#fbf9fe">交易记录</load-more>
      <x-table>
        <thead>
          <tr>
            <th>名称</th>
            <th>代码</th>
            <th>类型</th>
            <th>成交价</th>
            <th>数量</th>
            <th>时间</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="item in logs" :key="{item}">
            <td>{{item.name}}</td>
            <td>{{item.code}}</td>
            <td>{{item.opt}}</td>
            <td>{{item.price}}</td>
            <td>{{item.num}}</td>
            <td>{{item.ctime}}</td>
          </tr>
        </tbody>
      </x-table>
    </div>
  </div>
</template>

<script>
import {
  ButtonTab,
  ButtonTabItem,
  XTable,
  XButton,
  Flexbox,
  FlexboxItem
} from "vux";
export default {
  components: {
    ButtonTab,
    ButtonTabItem,
    XTable,
    XButton,
    Flexbox,
    FlexboxItem
  },

  data() {
    return {
      cuTab: "1",
      logs: [],
      interest: [],
      clearance: [],
      dataBase: null
    };
  },
  beforeCreate: function() {
    console.log("beforeCreate");
  },
  created: function() {
    console.log("created");
    this.initDb();
    this.initTable();
    this.getData();
  },
  beforeMount: function() {
    console.log("test");
  },
  mounted: function() {},
  beforeUpdate: function() {},
  updated: function() {},
  beforeDestroy: function() {
    console.log("test");
  },
  destroyed: function() {},
  methods: {
    initDb() {
      this.dataBase = window.openDatabase(
        "stock",
        window.localStorage.DBversion,
        "stock",
        1024 * 1024 * 1024
      );
      if (this.dataBase) {
        console.log("数据库创建/打开成功!");
      } else {
        console.log("数据库创建/打开失败！");
      }
    },
    initTable() {
      let sql =
        "CREATE TABLE IF NOT EXISTS stockLog(id integer primary key autoincrement,price INTEGER,name text,code text,num int,opt text,ctime text)";
      this.dataBase.transaction(function(ctx, result) {
        ctx.executeSql(sql);
      });
    },
    insertData(name, code, opt, price, num, ctime) {
      let values =
        "('" +
        name +
        "','" +
        code +
        "','" +
        opt +
        "','" +
        this.trim(price) +
        "'," +
        num +
        ",'" +
        ctime +
        "')";
      let sql =
        "INSERT INTO stockLog(name,code,opt,price,num,ctime) VALUES " + values;
      this.dataBase.transaction(function(ctx, result) {
        ctx.executeSql(sql);
      });
    },
    trim(str) {
      //删除左右两端的空格
      return str.replace(/(^\s*)|(\s*$)/g, "");
    },
    getData() {
      let that = this;
      this.dataBase.transaction(function(tx) {
        tx.executeSql(
          "SELECT * FROM stockLog",
          [],
          function(tx, results) {
            for (let i = 0; i < results.rows.length; i++) {
              let item = {};
              item = results.rows.item(i);
              that.logs.push(item);
            }
            that.totalInterest();
          },
          null
        );
      });
    },
    /*计算持仓*/
    totalInterest() {
      let numTotal = {};
      let priceTotal = {};
      let name = {};
      for (let i = 0; i < this.logs.length; i++) {
        let item = this.logs[i];
        name[item.code] = item.name;
        if (item.opt == "卖出") {
          numTotal[item.code] =
            (numTotal[item.code] || 0) - parseInt(item.num, 10);
          priceTotal[item.code] =
            (priceTotal[item.code] || 0) - parseInt(item.num, 10) * item.price;
        }
        if (item.opt == "买入") {
          numTotal[item.code] =
            (numTotal[item.code] || 0) + parseInt(item.num, 10);
          priceTotal[item.code] =
            (priceTotal[item.code] || 0) + parseInt(item.num, 10) * item.price;
        }
      }

      for (let i in numTotal) {
        //持仓的
        if (numTotal[i] > 0) {
          let buyPrice = priceTotal[i] / numTotal[i];
          buyPrice = Math.floor(buyPrice * 1000) / 1000;
          let item = {
            name: name[i],
            code: i,
            num: numTotal[i],
            buyPrice: buyPrice
          };
          this.interest.push(item);
        }
        //清仓的
        if (numTotal[i] == 0) {
          let profitLoss = Math.floor(Math.abs(priceTotal[i]) * 1) / 1;
          let item = {
            name: name[i],
            code: i,
            profitLoss: profitLoss
          };
          this.clearance.push(item);
        }
      }
      //get price
      this.showPrice();
      console.log("numTotal", numTotal);
      console.log("priceTotal", priceTotal);
    },
    /*通过新浪接口获取股价*/
    showPrice() {
      let list = "";
      for (let i = 0; i < this.interest.length; i++) {
        let code = this.interest[i].code.toLowerCase();
        list = list.length == 0 ? code : list + "," + code;
      }
      let url = "http://hq.sinajs.cn/list=" + list;
      var body = document.getElementsByTagName("body")[0];
      var jsNode = document.createElement("script");
      jsNode.setAttribute("type", "text/javascript");
      jsNode.setAttribute("src", url);
      body.appendChild(jsNode);
      let that = this;
      jsNode.onload = function() {
        console.log("window", window);

        for (let i = 0; i < that.interest.length; i++) {
          let code = that.interest[i].code;
          if (window["hq_str_" + code.toLowerCase()]) {
            let tmpPrice = window["hq_str_" + code.toLowerCase()].split(",");
            that.interest[i].price = tmpPrice[3];
            that.interest[i].market = tmpPrice[3] * that.interest[i].num;
            that.interest[i].market =
              Math.floor(that.interest[i].market * 1) / 1;
            that.interest[i].profitLoss =
              that.interest[i].market -
              that.interest[i].buyPrice * that.interest[i].num;
            that.interest[i].profitLoss =
              Math.floor(that.interest[i].profitLoss * 1) / 1;
          }
        }
        that.interest = JSON.parse(JSON.stringify(that.interest));
        console.log("interest", that.interest);
        // do something
      };
    },
    exCsv() {
      console.log("excsv");
      let filename = "交易记录.csv";
      const BOM = "\uFEFF";
      let header = "名称,代码,类型,成交价,数量,时间" + "\r\n";
      let csv = BOM + header;

      for (let i = 0; i < this.logs.length; i++) {
        let item = this.logs[i];
        let line =
          item.name +
          "," +
          item.code +
          "," +
          item.opt +
          "," +
          item.price +
          "," +
          item.num +
          "," +
          item.ctime +
          "\r\n";
        csv = csv + line;
      }
      let blob = new Blob([csv]);
      if (navigator.msSaveOrOpenBlob) {
        navigator.msSaveOrOpenBlob(blob, filename);
      } else {
        let url = URL.createObjectURL(blob);
        let downloadLink = document.createElement("a");
        downloadLink.href = url;
        downloadLink.download = filename;
        document.body.appendChild(downloadLink);
        downloadLink.click();
        document.body.removeChild(downloadLink);
        URL.revokeObjectURL(url);
      }
    },
    tirggerFile($event) {
      var file = event.target.files;
      if (window.FileReader) {
        var fr = new FileReader();
        fr.readAsText(file[0], "gb2312");
        fr.onload = e => {
          let lines = e.target.result.split("\n");
          for (let i = 0; i < lines.length; i++) {
            let line = lines[i].split(",");
            if (line[2] == "买入" || line[2] == "卖出") {
              //name, code, opt, price, num, ctime
              this.insertData(
                line[0],
                line[1],
                line[2],
                line[4],
                line[5],
                line[3]
              );
            }
          }
        };
      }
    }
  }
};
</script>
<style>
.flex-demo {
  margin-top: 5px;
}
.fileBut {
  position: relative;
}

.fileBut2 {
  width: 100%;
  text-align: center;
}

.file_input {
  width: 100%; /*因为file-input在部分浏览器中会自带一个输入框，需要双击才可以点击上传,放大后将其定位到div外面就好啦*/
  height: 100%;
  position: absolute;
  left: 0px;
  top: 0;
  z-index: 1;
  -moz-opacity: 0;
  -ms-opacity: 0;
  -webkit-opacity: 0;
  opacity: 0; /*css属性——opcity不透明度，取值0-1*/
  filter: alpha(
    opacity=0
  ); /*兼容IE8及以下--filter属性是IE特有的，它还有很多其它滤镜效果，而filter: alpha(opacity=0); 兼容IE8及以下的IE浏览器(如果你的电脑IE是8以下的版本，使用某些效果是可能会有一个允许ActiveX的提示,注意点一下就ok啦)*/
  cursor: pointer;
}
</style>
