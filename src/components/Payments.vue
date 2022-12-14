<template>
  <main>
    <div class="payment-container">
      <div class="payment-container__head">
        <div class="head__nav">
          <span>Pagos</span>
          <img src="../assets/arrow-down.svg" alt="open nav" />
        </div>
        <div class="head__options">
          <div
            v-if="!editing"
            class="options__edit"
            @click="
              () => {
                editing = !editing;
              }
            "
          >
            <span>Editar</span>
            <img src="../assets/edit-pencil.svg" alt="edit" />
          </div>
          <div v-else class="options__edit">
            <button
              type="button"
              @click="
                () => {
                  editing = !editing;
                }
              "
            >
              Guardar
            </button>
          </div>
          <div class="options__amount">
            <span class="options__amount--light">Por cobrar:</span>
            <span class="options__amount--strong"
              >{{ TOTALTOPAY }} {{ currency }}</span
            >
          </div>
        </div>
      </div>
      <div class="payment-container__dues" :key="dues.length">
        <div class="dues__node" v-for="due in dues" :key="due.id">
          <DueComponent
            :due="due"
            @updateDue="updateDues"
            @deleteDue="deleteDue"
            :editing="editing"
            :totalDue="TOTALTOPAY"
            @changePercentage="percentageChange"
            :key="due.percentage"
          ></DueComponent>
          <hr :class="`line ${due.status === 'paid' ? 'green' : 'blue'}`" />
        </div>

        <div class="payment-container__new-payment">
          <div class="circle" @click="createDue">
            <img src="../assets/add-plus.svg" alt="" />
          </div>
        </div>
      </div>
    </div>
  </main>
</template>

<script>
import DueComponent from "./Due.vue";
import { get, post } from "../utils/api.js";
export default {
  name: "PaymentsComponent",
  components: {
    DueComponent,
  },
  data() {
    return {
      dues: [],
      TOTALTOPAY: 182,
      editing: false,
      currency: "UF",
      quantityPaidDues: 0,
      totalPaid: 0,
    };
  },
  beforeMount() {
    const dues = get();
    if (dues) {
      this.dues = dues;
      this.dues.forEach((item) => {
        if (item.status === "paid") {
          this.totalPaid += item.percentage;
        }
      });
    }
  },
  methods: {
    updateDues(data) {
      /*
       Update dues in frontend UI and also in Local storage

       @todo: api request for update 
       */
      this.dues = this.dues.map((item) => {
        if (item.id === data.id) {
          if (item.status !== data.value.status) {
            this.totalPaid += data.value.percentage;
          }
          return { ...data.value };
        }

        return item;
      });

      post([...this.dues]);
    },
    calculateAmount() {
      /*
      Calculate amount and percentage for every new due
      */

      if (this.dues.length === 0) {
        return { amount: this.TOTALTOPAY, percentage: 100 };
      }
      const amount = parseFloat(
        (this.dues[this.dues.length - 1].amount / 2).toFixed(1)
      );
      const percentage = parseFloat(
        (this.dues[this.dues.length - 1].percentage / 2).toFixed(1)
      );

      return { amount, percentage };
    },
    createDue() {
      /*
       Create new Due in frontend UI and also in local storage

         @todo: api request to create new due
       */
      if (this.totalPaid === 100) {
        return;
      }

      this.editing = true;
      const { amount, percentage } = this.calculateAmount();

      this.dues.push({
        title: "Cuota",
        amount: amount,
        order: this.dues.length + 1,
        id: this.dues.slice(-1)[0]?.id + 1 || 1,
        percentage: percentage,
        date: new Date().toLocaleDateString(),
      });

      if (this.dues.length >= 2) {
        const idPrevious = this.dues.slice(-1)[0].id - 1;
        this.dues = this.dues.map((item) =>
          item.id === idPrevious ? { ...item, amount, percentage } : item
        );
      }
      post([...this.dues]);
    },
    deleteDue(id) {
      /*
      Delete due from frontend UI and also in localstorage. Also, change percentage and amount for previous due

      @todo: api request for delete
      */

      if (this.dues[0].id !== id) {
        const removedDue = this.dues.find((item) => item.id === id);
        let idToModify = id;
        this.dues = this.dues
          .filter((item) => item.id !== idToModify)
          .map((item) => {
            if (item.id === idToModify - 1 && item.status !== "paid") {
              item.percentage = item.percentage + removedDue.percentage;
              item.amount = parseFloat(
                ((item.percentage * this.TOTALTOPAY) / 100).toFixed(1)
              );
            } else if (item.id === id - 1 && item.status === "paid") {
              idToModify += 2;
            }

            return item;
          });

        this.dues = this.dues.map((item, idx) => {
          return { ...item, id: idx, order: idx };
        });
        post([...this.dues]);
      }
    },

    percentageChange(changeValue, dueID) {
      /*
      Special treatment when change percentage or amount for change this two
      in the Due which is being edited and also in the last due on the list
      to mantain the TOTALTOPAY on the large of the nodes

      @todo: api request
      */
      let duesLen = this.dues.length;

      if (duesLen === dueID) {
        duesLen = this.dues.length - 1;
      }

      const lastDueID = this.dues[duesLen - 1].id;

      const percentage = parseFloat(
        (this.dues[duesLen - 1].percentage + changeValue * -1).toFixed(1)
      );

      const amount = parseFloat(
        ((this.dues[duesLen - 1].percentage * this.TOTALTOPAY) / 100).toFixed(1)
      );

      this.dues = this.dues.map((item) =>
        item.id === lastDueID ? { ...item, amount, percentage } : item
      );

      post([...this.dues]);
    },
  },
};
</script>

<style scoped>
.payment-container {
  width: 100%;
}
.payment-container__head {
  display: flex;
  width: 100%;
  height: 80px;
  border-bottom: solid rgba(241, 245, 249, 1) 1px;
  justify-content: space-between;
  align-items: center;
}

.payment-container__head .head__nav {
  display: flex;
  margin-left: 20px;
  color: rgba(52, 96, 220, 1);
  font-weight: 600;
  font-size: 24px;
}

.payment-container__head .head__nav img {
  margin-left: 5px;
}

.payment-container__head .head__options {
  display: flex;
  margin-right: 20px;
  text-align: center;
}

.payment-container__head .head__options .options__edit {
  display: flex;
  margin-right: 15px;
  cursor: pointer;
}

.payment-container__head .head__options .options__edit span {
  color: rgba(52, 96, 220, 1);
  font-weight: 600;
  font-size: 16px;
}
.payment-container__head .head__options .options__edit img {
  width: 16px;
  height: 16px;
  margin-left: 5px;
}

.payment-container__head .head__options .options__edit button {
  width: 75px;
  height: 35px;
  background-color: rgba(29, 78, 216, 1);
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}

.payment-container__head .head__options .options__amount {
  display: flex;
  align-items: center;
}

.payment-container__head .head__options .options__amount--light {
  font-weight: 400;
  size: 24px;
  margin-right: 5px;
  color: rgba(148, 163, 184, 1);
}

.payment-container__head .head__options .options__amount--strong {
  font-weight: 600;
  size: 24px;
}

.payment-container__dues {
  position: relative;
  display: grid;
  align-items: flex-start;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  grid-auto-columns: minmax(200px, 1fr);
  grid-auto-flow: column;
  justify-items: flex-start;
  height: 100%;
  width: 100%;
  margin: 50px 0px 0px 0px;
  overflow-x: auto;
  white-space: nowrap;
  padding-bottom: 20px;
  padding-left: 5px;
}

.payment-container__dues .dues__node {
  width: 100%;
  position: relative;
}

.payment-container__dues .line {
  position: absolute;
  min-width: 100%;
  height: 2px;
  z-index: -1;
  top: 15%;
  left: 50%;
  border: none;
}

.payment-container__dues .green {
  background-color: rgba(5, 150, 105, 1);
}
.payment-container__dues .blue {
  background-color: rgba(29, 78, 216, 1);
}

.payment-container__new-payment {
  position: relative;
  display: flex;
  justify-content: center;
  width: 100%;
  height: 100%;
}

.payment-container__new-payment .line-new-payment {
  top: 15%;
}

.payment-container__new-payment .circle {
  display: flex;
  width: 50px;
  height: 50px;
  border-radius: 50%;
  background-color: grey;
  align-items: center;
  justify-content: center;
  background-color: rgba(226, 232, 240, 1);
  cursor: pointer;
}

.payment-container__new-payment .circle img {
  width: 20px;
  height: 20px;
}
</style>
