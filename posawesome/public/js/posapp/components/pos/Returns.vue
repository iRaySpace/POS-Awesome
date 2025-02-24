<template>
  <v-row justify="center">
    <v-dialog v-model="invoicesDialog" max-width="800px" min-width="800px">
      <v-card>
        <v-card-title>
          <span class="headline indigo--text">Select Return Invoice</span>
        </v-card-title>
        <v-container>
          <v-row class="mb-4">
            <v-text-field
              color="indigo"
              label="Invoice ID"
              background-color="white"
              hide-details
              v-model="invoice_name"
              dense
              clearable
              class="mx-4"
            ></v-text-field>
            <v-btn
              text
              class="ml-2"
              color="primary"
              dark
              @click="search_invoices"
              >Search</v-btn
            >
          </v-row>
          <v-row>
            <v-col cols="12" class="pa-1" v-if="dialog_data">
              <template>
                <v-data-table
                  :headers="headers"
                  :items="dialog_data"
                  item-key="name"
                  class="elevation-1"
                  :single-select="singleSelect"
                  show-select
                  v-model="selected"
                >
                  <template v-slot:item.grand_total="{ item }">{{
                    formtCurrency(item.grand_total)
                  }}</template>
                </v-data-table>
              </template>
            </v-col>
          </v-row>
        </v-container>
        <v-card-actions class="mt-4">
          <v-spacer></v-spacer>
          <v-btn color="error" dark @click="close_dialog">Close</v-btn>
          <v-btn
            v-if="selected.length"
            color="primary"
            dark
            @click="submit_dialog"
            >Select</v-btn
          >
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-row>
</template>

<script>
import { evntBus } from '../../bus';
export default {
  data: () => ({
    invoicesDialog: false,
    singleSelect: true,
    selected: [],
    dialog_data: '',
    company: '',
    invoice_name: '',
    headers: [
      {
        text: 'Customer',
        value: 'customer',
        align: 'start',
        sortable: true,
      },
      {
        text: 'Date',
        align: 'start',
        sortable: true,
        value: 'posting_date',
      },
      {
        text: 'Invoice',
        value: 'name',
        align: 'start',
        sortable: true,
      },
      {
        text: 'Amount',
        value: 'grand_total',
        align: 'start',
        sortable: false,
      },
    ],
  }),
  watch: {},
  methods: {
    close_dialog() {
      this.invoicesDialog = false;
    },
    search_invoices() {
      const vm = this;
      frappe.call({
        method: 'posawesome.posawesome.api.posapp.search_invoices_for_return',
        args: {
          invoice_name: vm.invoice_name,
          company: vm.company,
        },
        async: false,
        callback: function (r) {
          if (r.message) {
            vm.dialog_data = r.message;
          }
        },
      });
    },
    submit_dialog() {
      if (this.selected.length > 0) {
        const return_doc = this.selected[0];
        const invoice_doc = {};
        const items = [];
        return_doc.items.forEach((item) => {
          const new_item = { ...item };
          new_item.qty = item.qty * -1;
          new_item.stock_qty = item.stock_qty * -1;
          new_item.amount = item.amount * -1;
          items.push(new_item);
        });
        invoice_doc.items = items;
        invoice_doc.is_return = 1;
        invoice_doc.return_against = return_doc.name;
        invoice_doc.customer = return_doc.customer;
        const data = { invoice_doc, return_doc };
        evntBus.$emit('load_return_invoice', data);
        this.invoicesDialog = false;
      }
    },
    formtCurrency(value) {
      value = parseFloat(value);
      return value.toFixed(2).replace(/\d(?=(\d{3})+\.)/g, '$&,');
    },
  },
  created: function () {
    evntBus.$on('open_returns', (data) => {
      this.invoicesDialog = true;
      this.company = data;
      this.invoice_name = '';
      this.dialog_data = '';
      this.selected = [];
    });
  },
};
</script>