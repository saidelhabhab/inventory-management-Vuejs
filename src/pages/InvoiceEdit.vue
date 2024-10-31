<template>
  <div class="container new-invoice">
    <h1 class="text-center mb-4">Edit Invoice</h1>

    <!-- Edit Invoice Form -->
    <form @submit.prevent="handleSubmit" class="p-4 border rounded shadow">
      <div class="mb-3">
        <label for="client_id" class="form-label">Client:</label>
        <select v-model="form.client_id" class="form-select" id="client_id" required>
          <option disabled value="">Select a client</option>
          <option v-for="client in clients" :key="client.id" :value="client.id">
            {{ client.name }}
          </option>
        </select>
      </div>

      <div class="mb-3">
        <label for="status" class="form-label">Status:</label>
        <select v-model="form.status" class="form-select" id="status" required>
          <option disabled value="">Select status</option>
          <option value="paid">Paid</option>
          <option value="pending">Pending</option>
          <option value="canceled">Canceled</option>
        </select>
      </div>

      <!-- Additional Fields -->
      <div class="mb-3">
        <label for="factor_code" class="form-label">Factor Code:</label>
        <input type="text" v-model="form.factor_code" class="form-control" id="factor_code" required />
      </div>

      <div class="mb-3">
        <label for="due_date" class="form-label">Due Date:</label>
        <input type="date" v-model="form.due_date" class="form-control" id="due_date" required />
      </div>

      <div class="mb-3">
        <label for="payment_type" class="form-label">Payment Type:</label>
        <select v-model="form.payment_type" class="form-select" id="payment_type" required>
          <option disabled value="">Select payment type</option>
          <option value="check">Check</option>
          <option value="cash">Cash</option>
        </select>
      </div>


      <h5 class="mt-4">Invoice Items</h5>
      <table class="table table-bordered">
        <thead>
          <tr>
            <th>Product</th>
            <th>Price ($)</th>
            <th>Quantity</th>
            <th>Total ($)</th>
            <th>Actions</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(item, index) in form.invoice_items" :key="index">
            <td>
              <select v-model="item.product_id" class="form-select" @change="updatePrice(item)" required>
                <option disabled value="">Select a product</option>
                <option v-for="product in products" :key="product.id" :value="product.id">
                  {{ product.name }} (Q {{ product.quantity }})
                </option>
              </select>
            </td>
            <td>
              <span>{{ item.price}}</span>
            </td>
            <td>
              <input type="number" v-model.number="item.quantity" class="form-control" min="0" step="0.01" required />
            </td>
            <td>
              {{ calculateTotal(item.quantity, item.price) }}
            </td>
            <td>
              <button type="button" class="btn btn-sm btn-danger" @click="removeInvoiceItem(index)">Remove</button>
            </td>
          </tr>
        </tbody>
      </table>
      <button type="button" class="btn btn-secondary" @click="addInvoiceItem">Add Item</button>

      <!-- Debugging output 
      <pre>{{ form.items }}</pre>  Add this for debugging -->

      <!-- Amount (Automatically Calculated) -->
      <div class="mb-3">
        <label for="amount" class="form-label">Amount ($):</label>
        <input
          type="number"
          v-model.number="Amount"
          class="form-control"
          id="amount"
          readonly
        />
      </div>

      <div class="mb-3">
        <label for="tva" class="form-label">TVA (%):</label>
        <input type="number" v-model.number="form.tva" class="form-control" id="tva" min="0" step="0.01" required />
      </div>

      <!-- Total Amount (Automatically Calculated) -->
      <div class="mb-3">
        <label for="total-amount" class="form-label">Total Amount ($):</label>
        <input
          type="number"
          v-model.number="totalAmountWithTva"
          class="form-control"
          id="total-amount"
          readonly
        />
      </div>

      <div class="mb-3">
        <label for="final_price" class="form-label">Final Price ($):</label>
        <input type="number" v-model.number="form.final_price" class="form-control" id="final_price" min="0" step="0.01" required />
      </div>

      <div class="mb-3">
        <label for="remaining_price" class="form-label">Remaining Price ($):</label>
        <input type="number" v-model.number="form.remaining_price" class="form-control" id="remaining_price" min="0" step="0.01" required />
      </div>

      <button type="submit" class="btn btn-success mt-3">Save Changes</button>

      <p v-if="errorMessage" class="text-danger mt-3">{{ errorMessage }}</p>
    </form>
  </div>
</template>

<script>
import { ref, onMounted, computed } from 'vue';
import axios from '../services/axios';
import Swal from 'sweetalert2';
import { useRoute } from 'vue-router';

export default {
  name: 'EditInvoice',
  setup() {
    const clients = ref([]);
    const products = ref([]);
    
    const route = useRoute();
    const invoiceId = ref(route.params.id);

    // Initialize form
    const form = ref({
      client_id: '',
      amount: 0,
      total_amount_with_tva: 0,
      status: '',
      due_date: '',
      factor_code: '',
      final_price: 0,
      remaining_price: 0,
      payment_type: '',
      tva: 0,
      invoice_items: [], // Ensure this is an array
    });
    
    const errorMessage = ref('');

    // Fetch all clients from the API
    const fetchClients = async () => {
      try {
        const response = await axios.get('/clients');
        clients.value = response.data;
      } catch (error) {
        console.error('Error fetching clients:', error);
        Swal.fire('Error', 'Failed to fetch clients.', 'error');
      }
    };

    // Fetch all products from the API
    const fetchProducts = async () => {
      try {
        const response = await axios.get('/products');
        products.value = response.data;
      } catch (error) {
        console.error('Error fetching products:', error);
        Swal.fire('Error', 'Failed to fetch products.', 'error');
      }
    };

    // Fetch the invoice data by ID
    const fetchInvoiceData = async () => {
      try {
        const response = await axios.get(`/invoices/${invoiceId.value}`);
        form.value = { ...response.data, invoice_items: response.data.invoice_items || [] }; // Ensure invoice_items is an array

        console.log('response.data:', response.data, "items ==> ", response.data.invoice_items);
      } catch (error) {
        console.error('Error fetching invoice:', error);
        Swal.fire('Error', 'Failed to fetch invoice data.', 'error');
      }
    };

    // Add a new invoice item
    const addInvoiceItem = () => {
      form.value.invoice_items.push({
        product_id: '',
        quantity: 1,
        price: 0,
      });
    };

    // Remove an invoice item
    const removeInvoiceItem = (index) => {
      if (form.value.invoice_items.length > 1) {
        form.value.invoice_items.splice(index, 1);
      } else {
        Swal.fire('Warning', 'You must have at least one invoice item.', 'warning');
      }
    };

    // Calculate total for each item
    const calculateTotal = (quantity, price) => {
      return (quantity * price).toFixed(2);
    };

    // Update the price when a product is selected
    const updatePrice = (item) => {
      const selectedProduct = products.value.find(product => product.id === item.product_id);
      if (selectedProduct) {
        item.price = parseFloat(selectedProduct.price);
      } else {
        item.price = 0;
      }
    };

    // Calculate total amount without VAT
    const Amount = computed(() => {
      return form.value.invoice_items.reduce((amount, item) => {
        return amount + (item.quantity * item.price);
      }, 0).toFixed(2);
    });

    // Calculate total amount with VAT
    const totalAmountWithTva = computed(() => {
      const totalAmount = form.value.invoice_items.reduce((sum, item) => {
        return sum + (item.price * item.quantity);
      }, 0);
      const tvaAmount = totalAmount * (form.value.tva / 100);
      return (totalAmount + tvaAmount).toFixed(2);
    });

    // Submit form and update the invoice ////////////////////////////
    const handleSubmit = async () => {

        console.log('invoiceId.value:', invoiceId.value, "form.value ==> ", JSON.stringify(form.value));
      try {
        await axios.put(`/invoices/${invoiceId.value}`, form.value);
       
        Swal.fire('Success', 'Invoice updated successfully!', 'success');
      } catch (error) {
        console.error('Error updating invoice:', error);
        errorMessage.value = 'Failed to update invoice. Please check your input.';
      }
    };

    // Fetch data on component mount
    onMounted(() => {
      fetchClients();
      fetchProducts();
      fetchInvoiceData(); // No need to pass props.invoiceId
    });

    return {
      clients,
      products,
      form,
      errorMessage,
      addInvoiceItem,
      removeInvoiceItem,
      calculateTotal,
      updatePrice,
      Amount,
      totalAmountWithTva,
      handleSubmit,
    };
  },
};
</script>

<style scoped>
.new-invoice {
  max-width: 800px;
  margin: 0 auto;
}
</style>
