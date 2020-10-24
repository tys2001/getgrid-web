<template>
  <b-card class="m-2">
    <h1>GetGrid -SQL自動作成ツール-</h1>
    <p>
      表データからINSERT文などのSQLを自動生成するためのツールです。詳細な説明は<a
        href="https://blog.tyspine.com/getgrid/"
        >こちら</a
      >
    </p>
    <ol>
      <li>
        <p>
          まず、SQLに変換したい表をExcelなどからコピーしてください。必ずヘッダ行を含めてコピーしてください。
        </p>
      </li>
      <li>
        <p>
          次に、コピーした表をこちらに貼り付けます。成功すると、取り込んだ表が下に表示されます。
        </p>
        <b-input
          type="text"
          class="input-text"
          @paste.prevent="readTable"
          maxlength="0"
          placeholder="ここに貼り付け"
        />
        <b-table-simple v-if="rows.length > 0" sticky-header bordered outlined>
          <b-thead head-variant="dark">
            <b-tr>
              <b-th v-for="(header, i) in headers" :key="i">{{ header }}</b-th>
            </b-tr>
          </b-thead>
          <b-tbody>
            <b-tr v-for="(row, itr) in rows" :key="itr">
              <b-td v-for="(cell, itd) in row" :key="itd">{{ cell }}</b-td>
            </b-tr>
          </b-tbody>
        </b-table-simple>
        <b-alert v-else show variant="danger" class="alert"
          >テーブルが読み込まれていません。</b-alert
        >
      </li>

      <li>
        <p>テーブル名を入力あるいは選択してください。</p>
        <b-input
          type="text"
          class="input-text"
          v-model="tableName"
          placeholder="テーブル名を入力"
          list="table-name-cands"
        />
        <datalist id="table-name-cands">
          <option v-for="tableName in tableNameCands" :key="tableName">
            {{ tableName }}
          </option>
        </datalist>
        <b-alert v-if="!tableName" show variant="danger" class="alert"
          >テーブル名が入力されていません。</b-alert
        >
      </li>
      <li>
        <p>最後に、生成したいSQLの種類を選択すると、SQLが生成されます。</p>
        <b-button-group>
          <b-button @click="getInsert" variant="primary">INSERT文</b-button>
          <b-button @click="getUpdate" variant="primary">UPDATE文</b-button>
          <b-button @click="getDelete" variant="primary">DELETE文</b-button>
        </b-button-group>
        <div v-if="sql">
          SQLを生成しました。
          <b-button-group>
            <b-button @click="copySql" variant="primary">SQLをコピー</b-button>
            <b-button @click="downloadSql" variant="primary"
              >SQLをダウンロード</b-button
            >
          </b-button-group>
          <pre class="sql">{{ sql }}</pre>
        </div>
      </li>
    </ol>
  </b-card>
</template>

<script>
import { reactive, toRefs } from "@vue/composition-api";
export default {
  setup() {
    const state = reactive({
      headers: [],
      rows: [],
      tableName: "",
      sql: "",
      tableNameCands: JSON.parse(localStorage.tableNameCands || "[]"),
    });
    const methods = {
      readTable() {
        state.headers = [];
        state.rows = [];
        const text = event.clipboardData.getData("text/plain");
        const lines = text.split("\r\n");
        for (let line of lines) {
          const cells = line.split("\t");
          if (lines.indexOf(line) === 0) state.headers = cells;
          else state.rows.push(cells);
        }
      },
      getInsert() {
        if (state.rows.length === 0)
          return alert("テーブルのデータがありません。");
        if (!state.tableName) return alert("テーブル名が入力されていません。");
        state.sql = "";
        const headerSql = `INSERT INTO ${state.tableName} (${state.headers.join(
          ", "
        )}) VALUES`;
        for (let cells of state.rows) {
          state.sql += `${headerSql} ('${cells.join("', '")}')\r\n`;
        }
        methods.rememberTableName();
      },
      getUpdate() {
        if (state.rows.length === 0)
          return alert("テーブルのデータがありません。");
        if (!state.tableName) return alert("テーブル名が入力されていません。");
        const blankColIndex = state.headers.indexOf("");
        if (blankColIndex === -1)
          return alert("UPDATEの生成には空白列が必要です");
        state.sql = "";
        for (let cells of state.rows) {
          let setFormuras = [];
          let whereFormuras = [];
          for (let i in cells) {
            if (i < blankColIndex) {
              setFormuras.push(`${state.headers[i]}='${cells[i]}'`);
            }
            if (i > blankColIndex) {
              whereFormuras.push(`${state.headers[i]}='${cells[i]}'`);
            }
          }
          state.sql += `UPDATE ${state.tableName} SET ${setFormuras.join(
            ", "
          )} WHERE ${whereFormuras.join(" AND ")}\r\n`;
        }
        methods.rememberTableName();
      },
      getDelete() {
        if (state.rows.length === 0)
          return alert("テーブルのデータがありません。");
        if (!state.tableName) return alert("テーブル名が入力されていません。");
        state.sql = "";
        for (let cells of state.rows) {
          let whereFormuras = [];
          for (let i in cells) {
            whereFormuras.push(`${state.headers[i]}='${cells[i]}'`);
          }
          state.sql += `DELETE ${state.tableName} WHERE ${whereFormuras.join(
            " AND "
          )}\r\n`;
        }
        methods.rememberTableName();
      },
      copySql() {
        navigator.clipboard.writeText(state.sql);
      },
      downloadSql() {
        const blob = new Blob([state.sql], { type: "text/plain" });
        const atag = document.createElement("a");
        atag.href = URL.createObjectURL(blob);
        atag.setAttribute("download", "getgrid.txt");
        atag.dispatchEvent(new MouseEvent("click"));
      },
      rememberTableName() {
        if (!state.tableNameCands.includes(state.tableName)) {
          state.tableNameCands.push(state.tableName);
          state.tableNameCands.sort();
          localStorage.tableNameCands = JSON.stringify(state.tableNameCands);
        }
      },
    };
    return {
      ...toRefs(state),
      ...methods,
    };
  },
};
</script>

<style scoped>
h1,
li {
  margin-bottom: 30px;
}
li > * {
  margin-bottom: 10px;
}
.input-text {
  max-width: 200px;
}
.alert {
  max-width: 600px;
}
.sql {
  background-color: #dfd;
  padding: 20px;
  border-radius: 10px;
  max-height: 300px;
  margin: 10px 0;
}
</style>
