<!-- eslint-disable vue/multi-word-component-names -->
<template>
	<v-dialog v-model="showModal" max-width="95vw" max-height="95vh" persistent :fullscreen="$vuetify.display.mobile" attach="body">
		<v-card class="payments-modal">
				<v-card-title class="d-flex align-center px-4 py-3">
					<v-icon class="mr-3">mdi-credit-card</v-icon>
					<span class="text-h6">{{ __("Payment") }}</span>
					<v-spacer></v-spacer>
					<v-btn icon="mdi-close" variant="text" class="icon-close-btn" @click="back_to_invoice"></v-btn>
			</v-card-title>
			<v-divider class="payments-divider"></v-divider>
			<v-card-text class="pa-0">
				<div class="pa-0">
					<v-card class="selection payments-card mx-auto my-0 mt-1 pos-themed-card">
						<v-progress-linear :active="loading" :indeterminate="loading" absolute location="top" color="info"></v-progress-linear>
						<div ref="paymentContainer" class="payments-body">
							<div v-if="invoice_doc" class="payment-layout">
								<section class="payment-section payment-section--summary">
									<div class="payment-section__header">
										<v-icon size="small" class="payment-section__icon">mdi-finance</v-icon>
										<span class="payment-section__title">{{ __("Payment Summary") }}</span>
									</div>
									<v-row class="payment-section__content" dense>
										<v-col cols="12" md="6" lg="4">
											<v-text-field variant="solo" color="primary" :label="frappe._('Paid Amount')" class="sleek-field pos-themed-input read-only-field" hide-details :model-value="total_payments_display" readonly :prefix="currencySymbol(invoice_doc.currency)" density="compact" @click="showPaidAmount"></v-text-field>
										</v-col>
										<v-col cols="12" md="6" lg="4">
											<v-text-field variant="solo" color="primary" :label="diff_label" class="sleek-field pos-themed-input read-only-field" hide-details :model-value="diff_payment_display" readonly :prefix="currencySymbol(invoice_doc.currency)" density="compact" @click="showDiffPayment"></v-text-field>
										</v-col>
										<v-col cols="12" md="6" lg="4">
											<v-text-field variant="solo" color="primary" :label="frappe._('Discount')" class="sleek-field pos-themed-input read-only-field" hide-details :model-value="paymentDiscountDisplay" readonly :prefix="currencySymbol(invoice_doc.currency)" density="compact"></v-text-field>
										</v-col>
										<v-col cols="12" md="6" lg="4">
											<v-text-field variant="solo" color="primary" :label="frappe._('VAT Total')" class="sleek-field pos-themed-input read-only-field" hide-details :model-value="paymentTaxDisplay" readonly :prefix="currencySymbol(invoice_doc.currency)" density="compact"></v-text-field>
										</v-col>
										<v-col cols="12" md="6" lg="4">
											<v-text-field variant="solo" color="primary" :label="frappe._('Grand Total')" class="sleek-field pos-themed-input read-only-field" hide-details :model-value="invoiceTotalDisplay" readonly :prefix="currencySymbol(invoice_doc.currency)" density="compact"></v-text-field>
										</v-col>
									</v-row>
								</section>
								<section class="payment-section payment-section--methods" v-if="showPaymentMethodsSection">
									<div class="payment-section__header">
										<v-icon size="small" class="payment-section__icon">mdi-credit-card-multiple</v-icon>
										<span class="payment-section__title">{{ __("Payment Methods") }}</span>
									</div>
									<div class="payment-method-list">
										<div v-for="payment in invoice_doc.payments" :key="payment.name || payment.idx" class="payment-method-card">
											<div class="payment-method-card__header">
												<div class="payment-method-card__title">
													<span>{{ payment.mode_of_payment }}</span>
													<v-chip v-if="payment.default === 1" size="x-small" color="primary" class="ml-2">
														{{ __("Default") }}
													</v-chip>
													<v-chip v-if="payment.type === 'Phone'" size="x-small" color="info" class="ml-2">
														{{ __("Phone") }}
													</v-chip>
													<v-chip v-if="isMpesaPayment(payment)" size="x-small" color="success" class="ml-2">
														{{ __("M-Pesa") }}
													</v-chip>
												</div>
												<div class="payment-method-card__meta">
													<span v-if="payment.type">{{ __(payment.type) }}</span>
													<span v-else-if="payment.account">{{ payment.account }}</span>
												</div>
											</div>
											<v-row class="payment-method-card__content" dense>
												<v-col cols="12" md="4">
												<v-text-field
													v-if="payment.type !== 'Phone'"
													variant="outlined"
													color="primary"
													:label="frappe._('Amount')"
													class="pos-themed-input editable-field"
													hide-details
													density="compact"
													:prefix="currencySymbol(displayCurrency)"
													type="number"
													inputmode="decimal"
													v-model="payment.amount"
													:append-inner-icon="'mdi-dialpad'"
													@click:append-inner="openPaymentAmountKeypad(payment)"
													@focus="handlePaymentAmountFocus(payment)"
													@blur="normalizePaymentAmount(payment)"
												></v-text-field>
													<v-text-field v-else variant="solo" color="primary" :label="frappe._('Amount')" class="sleek-field pos-themed-input read-only-field" hide-details density="compact" :model-value="formatCurrency(payment.amount || 0)" :prefix="currencySymbol(displayCurrency)" readonly></v-text-field>
												</v-col>
												<v-col cols="12" md="8">
													<div class="payment-method-card__actions">
														<v-btn variant="tonal" size="small" color="primary" @click="set_rest_amount(payment)" :disabled="(!invoice_doc || !invoice_doc.is_return) && diff_payment <= 0">
															<v-icon size="small" class="mr-1">mdi-target</v-icon>
															{{ __("Set Remaining") }}
														</v-btn>
														<v-btn variant="tonal" size="small" color="secondary" @click="set_full_amount(payment)" :disabled="!canSetFullAmount">
															<v-icon size="small" class="mr-1">mdi-cash</v-icon>
															{{ __("Full Amount") }}
														</v-btn>
														<v-btn variant="text" size="small" color="grey-darken-1" @click="clearPaymentAmount(payment)">
															<v-icon size="small" class="mr-1">mdi-eraser</v-icon>
															{{ __("Clear") }}
														</v-btn>
														<v-btn v-if="payment.type === 'Phone'" variant="outlined" size="small" color="info" @click="openPhoneDialog(payment)">
															<v-icon size="small" class="mr-1">mdi-phone</v-icon>
															{{ __("Request Payment") }}
														</v-btn>
														<v-btn v-if="isMpesaPayment(payment)" variant="outlined" size="small" color="success" @click="mpesa_c2b_dialog(payment)">
															<v-icon size="small" class="mr-1">mdi-cellphone</v-icon>
															{{ __("Collect via M-Pesa") }}
														</v-btn>
													</div>
												</v-col>
											</v-row>
										</div>
									</div>
								</section>
								<section class="payment-section payment-section--change" v-if="showChangeSection">
									<div class="payment-section__header">
										<v-icon size="small" class="payment-section__icon">mdi-swap-horizontal</v-icon>
										<span class="payment-section__title">{{ __("Change & Adjustments") }}</span>
									</div>
									<v-row class="payment-section__content" dense>
										<v-col cols="12" md="4">
											<v-text-field
												variant="outlined"
												color="primary"
												:label="frappe._('Paid Change')"
												class="pos-themed-input editable-field"
												hide-details
												density="compact"
												:prefix="currencySymbol(displayCurrency)"
												type="number"
												inputmode="decimal"
												v-model="paid_change"
												:rules="paid_change_rules"
												:append-inner-icon="'mdi-dialpad'"
												@click:append-inner="openPaidChangeKeypad"
												@focus="onPaidChangeFocus"
												@blur="onPaidChangeBlur"
											></v-text-field>
										</v-col>
										<v-col cols="12" md="4">
											<v-text-field variant="solo" color="primary" :label="frappe._('Credit Change')" class="sleek-field pos-themed-input read-only-field" hide-details density="compact" :prefix="currencySymbol(displayCurrency)" :model-value="formatCurrency(credit_change || 0)" readonly></v-text-field>
										</v-col>
										<v-col cols="12" md="4">
											<v-text-field variant="solo" color="primary" :label="frappe._('Outstanding Balance')" class="sleek-field pos-themed-input read-only-field" hide-details density="compact" :model-value="formatCurrency(Math.max(diff_payment, 0))" :prefix="currencySymbol(displayCurrency)" readonly></v-text-field>
										</v-col>
									</v-row>
									<div class="payment-toggle-row" v-if="showChangeToggles">
										<v-switch v-if="showCreditSaleToggle" v-model="is_credit_sale" color="primary" :label="__('Mark as Credit Sale')" class="payment-toggle-row__switch"></v-switch>
										<v-switch v-if="invoice_doc.is_return" v-model="is_cashback" color="primary" :label="__('Give Change as Cash')" class="payment-toggle-row__switch"></v-switch>
										<v-switch v-if="invoice_doc.is_return" v-model="is_credit_return" color="primary" :label="__('Issue Credit Note')" class="payment-toggle-row__switch"></v-switch>
									</div>
								</section>
								<section class="payment-section payment-section--loyalty" v-if="showRewardsSection">
									<div class="payment-section__header">
										<v-icon size="small" class="payment-section__icon">mdi-star-circle</v-icon>
										<span class="payment-section__title">{{ __("Rewards & Credits") }}</span>
									</div>
									<v-row v-if="customer_info && customer_info.loyalty_points !== undefined" class="payment-section__content" dense>
										<v-col cols="12" md="6">
											<v-text-field
												variant="outlined"
												color="primary"
												:label="frappe._('Redeem Loyalty Amount')"
												class="pos-themed-input editable-field"
												hide-details
												density="compact"
												type="number"
												inputmode="decimal"
												v-model="loyalty_amount"
												:prefix="currencySymbol(displayCurrency)"
												:hint="__('Available: {0}', [formatCurrency(available_points_amount || 0)])"
												persistent-hint
												:append-inner-icon="'mdi-dialpad'"
												@click:append-inner="openLoyaltyKeypad"
											></v-text-field>
										</v-col>
									</v-row>
									<div v-if="pos_profile.use_customer_credit" class="payment-credit-toggle">
										<v-switch v-model="redeem_customer_credit" color="primary" :label="__('Redeem Customer Credit')"></v-switch>
									</div>
									<div v-if="redeem_customer_credit" class="customer-credit-list">
										<div v-for="(credit, index) in customer_credit_dict" :key="credit.name || `${credit.credit_origin}-${index}`" class="customer-credit-card">
											<div class="customer-credit-card__header">
												<strong>{{ credit.credit_origin || __('Advance') }}</strong>
												<span>{{ __('Available') }}: {{ formatCurrency(credit.total_credit || 0) }}</span>
											</div>
											<v-text-field
												variant="outlined"
												color="primary"
												:label="frappe._('Amount to Redeem')"
												class="pos-themed-input editable-field"
												hide-details
												density="compact"
												:prefix="currencySymbol(displayCurrency)"
												type="number"
												inputmode="decimal"
												v-model="credit.credit_to_redeem"
												:append-inner-icon="'mdi-dialpad'"
												@click:append-inner="openCustomerCreditKeypad(credit)"
												@blur="normalizeCustomerCredit(credit)"
											></v-text-field>
										</div>
										<div v-if="!customer_credit_dict.length" class="customer-credit-empty">
											{{ __("No customer credit available") }}
										</div>
									</div>
								</section>
								<section class="payment-section payment-section--details" v-if="showAdditionalDetailsSection">
									<div class="payment-section__header">
										<v-icon size="small" class="payment-section__icon">mdi-account-details</v-icon>
										<span class="payment-section__title">{{ __("Additional Details") }}</span>
									</div>
									<v-row class="payment-section__content" dense>
										<v-col v-if="showSalesPersonField" cols="12" md="6" lg="4">
											<v-autocomplete variant="outlined" color="primary" :label="frappe._('Sales Person')" class="pos-themed-input editable-field" hide-details density="compact" :items="sales_persons" item-title="title" item-value="value" clearable v-model="sales_person"></v-autocomplete>
										</v-col>
										<v-col v-if="showDeliveryDateField" cols="12" md="6" lg="4">
											<v-text-field variant="outlined" color="primary" :label="frappe._('Delivery Date')" class="pos-themed-input editable-field" hide-details density="compact" type="date" :model-value="formatDateInput(new_delivery_date || invoice_doc.posa_delivery_date)" @update:model-value="handleDeliveryDate"></v-text-field>
										</v-col>
										<v-col v-if="showCreditDueDateField" cols="12" md="6" lg="4">
											<v-text-field variant="outlined" color="primary" :label="frappe._('Purchase Order Date')" class="pos-themed-input editable-field" hide-details density="compact" type="date" :model-value="formatDateInput(new_po_date || invoice_doc.po_date)" @update:model-value="handlePODate"></v-text-field>
										</v-col>
										<v-col v-if="showCreditDueDateField" cols="12" md="6" lg="4">
											<v-text-field variant="outlined" color="primary" :label="frappe._('Credit Due Date')" class="pos-themed-input editable-field" hide-details density="compact" type="date" :model-value="formatDateInput(new_credit_due_date || invoice_doc.due_date)" @update:model-value="handleCreditDueDate"></v-text-field>
											<div class="credit-presets" v-if="credit_due_presets && credit_due_presets.length">
												<v-chip v-for="preset in credit_due_presets" :key="`preset-${preset}`" size="small" variant="outlined" color="primary" class="mr-2 mb-2" @click="applyDuePreset(preset)">
													{{ preset }} {{ __("Days") }}
												</v-chip>
												<v-chip size="small" variant="outlined" color="secondary" class="mb-2" @click="custom_days_dialog = true">
													{{ __("Custom") }}
												</v-chip>
											</div>
										</v-col>
										<v-col v-if="showShippingAddressField" cols="12" md="6" lg="8">
											<v-autocomplete variant="outlined" color="primary" :items="addressOptions" :item-title="item => item.label" :item-value="item => item.value" :label="frappe._('Shipping Address')" class="pos-themed-input editable-field" hide-details density="compact" v-model="invoice_doc.shipping_address_name" clearable></v-autocomplete>
											<v-btn variant="text" size="small" color="primary" class="mt-2" @click="new_address">
												<v-icon size="small" class="mr-1">mdi-plus-circle</v-icon>
												{{ __("Add Address") }}
											</v-btn>
										</v-col>
									</v-row>
								</section>
							</div>
							<div v-else class="payment-empty-state">
								<v-icon size="large" class="mb-2">mdi-information-outline</v-icon>
								<div>{{ __("Select or create an invoice to configure payments") }}</div>
							</div>
						</div>
					</v-card>
					<!-- Action Buttons -->
					<v-card flat class="cards payments-actions mb-0 mt-3">
						<v-row class="payments-actions-row" align="stretch" dense>
							<v-col cols="12" md="4">
								<v-btn ref="submitButton" block size="large" color="primary" theme="dark" class="submit-btn payments-actions-btn" @click="submit" :loading="loading" :disabled="loading || vaildatPayment" :class="{ 'submit-highlight': highlightSubmit }">
									{{ __("Submit") }}
								</v-btn>
							</v-col>
							<v-col cols="12" md="4">
								<v-btn block size="large" color="success" theme="dark" class="payment-action-btn payments-actions-btn" @click="submit(undefined, false, true)" :loading="loading" :disabled="loading || vaildatPayment">
									{{ __("Submit & Print") }}
								</v-btn>
							</v-col>
							<v-col cols="12" md="4">
								<v-btn block size="large" color="error" theme="dark" class="payment-action-btn payments-actions-btn" @click="back_to_invoice">
									{{ __("Cancel Payment") }}
								</v-btn>
							</v-col>
						</v-row>
					</v-card>
					<!-- Custom Days Dialog -->
					<v-dialog v-model="custom_days_dialog" max-width="300px" attach="body">
						<v-card>
							<v-card-title class="text-h6">
								{{ __("Custom Due Days") }}
							</v-card-title>
							<v-card-text class="pa-0">
								<v-container>
									<v-text-field density="compact" variant="solo" type="number" inputmode="numeric" min="0" max="365" class="sleek-field pos-themed-input" v-model.number="custom_days_value" :label="frappe._('Days')" hide-details></v-text-field>
								</v-container>
							</v-card-text>
							<v-card-actions>
								<v-spacer></v-spacer>
								<v-btn color="error" theme="dark" class="payment-action-btn" @click="custom_days_dialog = false">
									{{ __("Close") }}
								</v-btn>
								<v-btn color="primary" theme="dark" class="payment-action-btn" @click="applyCustomDays">
									{{ __("Apply") }}
								</v-btn>
							</v-card-actions>
						</v-card>
					</v-dialog>

					<!-- Phone Payment Dialog -->
					<v-dialog v-model="phone_dialog" max-width="400px" attach="body">
						<v-card>
							<v-card-title>
								<span class="text-h5 text-primary">{{ __("Confirm Mobile Number") }}</span>
							</v-card-title>
							<v-card-text class="pa-0">
								<v-container>
									<v-text-field density="compact" variant="solo" color="primary" :label="frappe._('Mobile Number')" class="sleek-field pos-themed-input" hide-details v-model="invoice_doc.contact_mobile" type="tel" inputmode="tel"></v-text-field>
								</v-container>
							</v-card-text>
							<v-card-actions>
								<v-spacer></v-spacer>
								<v-btn color="error" theme="dark" class="payment-action-btn" @click="phone_dialog = false">
									{{ __("Close") }}
								</v-btn>
								<v-btn color="primary" theme="dark" class="payment-action-btn" @click="request_payment">
									{{ __("Request") }}
								</v-btn>
							</v-card-actions>
						</v-card>
					</v-dialog>
				</div>
			</v-card-text>
		</v-card>
		<NumericKeypad
			:visible="numericKeypad.visible"
			:model-value="numericKeypad.value"
			:title="numericKeypad.title"
			:helper-text="numericKeypad.helperText"
			:allow-decimal="numericKeypad.allowDecimal"
			:allow-negative="numericKeypad.allowNegative"
			:decimal-places="numericKeypad.decimalPlaces"
			@update:modelValue="(val) => (numericKeypad.value = val)"
			@update:visible="(val) => (numericKeypad.visible = val)"
			@confirm="handleNumericKeypadConfirm"
			@cancel="closeNumericKeypad"
		/>
	</v-dialog>
</template>

<script>
/* global frappe, __, get_currency_symbol */
// Importing format mixin for currency and utility functions
import format, { formatUtils } from "../../format";
import {
	saveOfflineInvoice,
	syncOfflineInvoices,
	getPendingOfflineInvoiceCount,
	isOffline,
	getSalesPersonsStorage,
	setSalesPersonsStorage,
	updateLocalStock,
} from "../../../offline/index.js";

import renderOfflineInvoiceHTML from "../../../offline_print_template";
import { silentPrint } from "../../plugins/print.js";
import NumericKeypad from "./NumericKeypad.vue";

export default {
	// Using format mixin for shared formatting methods
	components: { NumericKeypad },
	mixins: [format],
	data() {
		return {
			showModal: false, // Modal visibility state
			loading: false, // UI loading state
			pos_profile: "", // POS profile settings
			pos_settings: "", // POS settings
			invoice_doc: "", // Current invoice document
			stock_settings: "", // Stock settings
			invoiceType: "Invoice", // Type of invoice
			is_return: false, // Is this a return invoice?
			loyalty_amount: 0, // Loyalty points to redeem
			redeemed_customer_credit: 0, // Customer credit to redeem
			credit_change: 0, // Change to be given as credit
			paid_change: 0, // Change to be given as paid
			is_credit_sale: false, // Is this a credit sale?
			is_write_off_change: false, // Write-off for change enabled
			is_cashback: true, // Cashback enabled
			is_credit_return: false, // Is this a credit return?
			redeem_customer_credit: false, // Redeem customer credit?
			customer_credit_dict: [], // List of available customer credits
			paid_change_rules: [], // Validation rules for paid change
			phone_dialog: false, // Show phone payment dialog
			custom_days_dialog: false, // Show custom days dialog
			custom_days_value: null, // Custom days entry
			new_delivery_date: null, // New delivery date value
			new_po_date: null, // New PO date value
			new_credit_due_date: null, // New credit due date value
			credit_due_days: null, // Number of days until due
			credit_due_presets: [7, 14, 30], // Preset options for due days
			customer_info: "", // Customer info
			mpesa_modes: [], // List of available M-Pesa modes
			sales_persons: [], // List of sales persons
			sales_person: "", // Selected sales person
			addresses: [], // List of customer addresses
			is_user_editing_paid_change: false, // User interaction flag
			highlightSubmit: false, // Highlight state for submit button
			prefersTouchKeypad: false,
			numericKeypad: {
				visible: false,
				value: "0",
				title: "",
				helperText: "",
				allowDecimal: true,
				allowNegative: false,
				decimalPlaces: 2,
				apply: null,
			},
		};
	},
	computed: {
		// Get currency symbol for given or current currency
		currencySymbol() {
			return (currency) => {
				return get_currency_symbol(currency || this.invoice_doc.currency);
			};
		},
		// Display currency for invoice
		displayCurrency() {
			return this.invoice_doc ? this.invoice_doc.currency : "";
		},
		isRoundedTotalDisabled() {
			const flag = this.pos_profile?.disable_rounded_total;
			if (flag === undefined || flag === null) {
				return false;
			}
			if (typeof flag === "boolean") {
				return flag;
			}
			if (typeof flag === "string") {
				return flag === "1" || flag.toLowerCase() === "true";
			}
			return Number(flag) === 1;
		},
		invoiceTotalRaw() {
			return this.resolveInvoiceTotal(this.invoice_doc);
		},
		invoiceTotal() {
			return this.flt(this.invoiceTotalRaw, this.currency_precision);
		},
		invoiceTotalDisplay() {
			return this.formatCurrency(this.invoiceTotal, this.displayCurrency);
		},
		blockSaleBeyondAvailableQty() {
			return (
				!["Order", "Quotation"].includes(this.invoiceType) &&
				this.pos_profile.posa_block_sale_beyond_available_qty
			);
		},
		// Calculate total payments (all methods, loyalty, credit)
		total_payments() {
			let total = 0;
			if (this.invoice_doc && this.invoice_doc.payments) {
				this.invoice_doc.payments.forEach((payment) => {
					// Payment amount is already in selected currency
					total += parseFloat(formatUtils.fromArabicNumerals(String(payment.amount))) || 0;
				});
			}

			// Add loyalty amount (convert if needed)
			if (this.loyalty_amount) {
				// Loyalty points are stored in base currency (PKR)
				if (this.invoice_doc.currency !== this.pos_profile.currency) {
					// Convert to selected currency (e.g. USD) by dividing
					total += this.flt(
						this.loyalty_amount / (this.invoice_doc.conversion_rate || 1),
						this.currency_precision,
					);
				} else {
					total += parseFloat(formatUtils.fromArabicNumerals(String(this.loyalty_amount))) || 0;
				}
			}

			// Add redeemed customer credit (convert if needed)
			if (this.redeemed_customer_credit) {
				// Customer credit is stored in base currency (PKR)
				if (this.invoice_doc.currency !== this.pos_profile.currency) {
					// Convert to selected currency (e.g. USD) by dividing
					total += this.flt(
						this.redeemed_customer_credit / (this.invoice_doc.conversion_rate || 1),
						this.currency_precision,
					);
				} else {
					total +=
						parseFloat(formatUtils.fromArabicNumerals(String(this.redeemed_customer_credit))) ||
						0;
				}
			}

			return this.flt(total, this.currency_precision);
		},

		// Calculate difference between invoice total and payments
		diff_payment() {
			if (!this.invoice_doc) return 0;

			const invoice_total = this.invoiceTotal;
			let diff = this.flt(invoice_total - this.total_payments, this.currency_precision);

			if (this.invoice_doc.is_return) {
				return diff >= 0 ? diff : 0;
			}

			return diff >= 0 ? diff : 0;
		},

		// Calculate change to be given back to customer
		credit_change() {
			const invoice_total = this.invoiceTotal;
			let change = this.flt(this.total_payments - invoice_total, this.currency_precision);
			return change > 0 ? change : 0;
		},

		// Label for the difference field (To Be Paid/Change)
		diff_label() {
			return this.diff_payment > 0
				? `To Be Paid (${this.displayCurrency})`
				: `Change (${this.displayCurrency})`;
		},
		// Display formatted total payments
		total_payments_display() {
			return this.formatCurrency(this.total_payments, this.displayCurrency);
		},
		// Display formatted difference payment
		diff_payment_display() {
			return this.formatCurrency(this.diff_payment, this.displayCurrency);
		},
		// Display formatted tax total for summary
		paymentTaxTotal() {
			if (!this.invoice_doc) {
				return 0;
			}
			let taxTotal = this.invoice_doc.total_taxes_and_charges || 0;
			if (
				this.pos_profile?.posa_allow_multi_currency &&
				this.invoice_doc.currency !== this.pos_profile.currency
			) {
				taxTotal = taxTotal / (this.invoice_doc.conversion_rate || 1);
			}
			return this.flt(taxTotal, this.currency_precision);
		},
		paymentTaxDisplay() {
			return this.formatCurrency(this.paymentTaxTotal, this.displayCurrency);
		},
		paymentDiscountTotal() {
			if (!this.invoice_doc) {
				return 0;
			}
			let discount = parseFloat(formatUtils.fromArabicNumerals(String(this.invoice_doc.discount_amount || 0)));
			if (Number.isNaN(discount)) {
				discount = 0;
			}
			if (this.invoice_doc.is_return) {
				discount = Math.abs(discount);
			}
			return this.flt(discount, this.currency_precision);
		},
		paymentDiscountDisplay() {
			return this.formatCurrency(this.paymentDiscountTotal, this.displayCurrency);
		},
		// Calculate available loyalty points amount in selected currency
		available_points_amount() {
			let amount = 0;
			if (this.customer_info.loyalty_points) {
				// Convert loyalty points to amount in base currency (PKR)
				amount = this.customer_info.loyalty_points * this.customer_info.conversion_factor;

				// Convert to selected currency if needed
				if (this.invoice_doc.currency !== this.pos_profile.currency) {
					// Convert PKR to USD by dividing
					amount = this.flt(
						amount / (this.invoice_doc.conversion_rate || 1),
						this.currency_precision,
					);
				}
			}
			return amount;
		},
		// Calculate total available customer credit
		available_customer_credit() {
			return this.customer_credit_dict.reduce((total, row) => total + this.flt(row.total_credit), 0);
		},
		// Validate if payment can be submitted
		vaildatPayment() {
			if (this.pos_profile.posa_allow_sales_order) {
				if (this.invoiceType === "Order" && !this.invoice_doc.posa_delivery_date) {
					return true;
				}
			}
			return false;
		},
		// Should request payment field be shown?
		request_payment_field() {
			return (
				this.pos_settings?.invoice_fields?.some(
					(el) => el.fieldtype === "Button" && el.fieldname === "request_for_payment",
				) || false
			);
		},
		addressOptions() {
			if (!Array.isArray(this.addresses)) {
				return [];
			}
			return this.addresses.map((address) => {
				const parts = [address.address_title, address.address_line1, address.city]
					.filter(Boolean)
					.map((part) => String(part));
				return {
					value: address.name,
					label: parts.length ? parts.join(" • ") : address.name,
				};
			});
		},
		showPaymentMethodsSection() {
			if (!this.invoice_doc || !Array.isArray(this.invoice_doc.payments)) {
				return false;
			}
			return ["Invoice", "Return"].includes(this.invoiceType) && this.invoice_doc.payments.length > 0;
		},
		showChangeSection() {
			if (!this.invoice_doc) {
				return false;
			}
			return ["Invoice", "Return"].includes(this.invoiceType) && (this.invoice_doc.payments || []).length > 0;
		},
		showRewardsSection() {
			if (!this.invoice_doc) {
				return false;
			}
			const hasLoyaltyInfo = this.customer_info && this.customer_info.loyalty_points !== undefined;
			const canRedeemCredit = this.pos_profile && this.pos_profile.use_customer_credit;
			return ["Invoice", "Return"].includes(this.invoiceType) && (hasLoyaltyInfo || canRedeemCredit);
		},
		showOrderDetails() {
			return this.invoice_doc && this.invoiceType === "Order";
		},
		showSalesPersonField() {
			return this.showOrderDetails || this.is_credit_sale;
		},
		showDeliveryDateField() {
			return this.showOrderDetails || this.is_credit_sale;
		},
		showCreditDueDateField() {
			if (!this.invoice_doc) {
				return false;
			}
			return this.invoiceType === "Order" || this.is_credit_sale;
		},
		showShippingAddressField() {
			return this.showOrderDetails && this.invoice_doc && !!this.invoice_doc.customer;
		},
		showAdditionalDetailsSection() {
			if (!this.invoice_doc) {
				return false;
			}
			return this.showOrderDetails || this.is_credit_sale || this.showShippingAddressField || this.showCreditDueDateField;
		},
		showCreditSaleToggle() {
			return this.invoiceType === "Invoice" && this.invoice_doc && !this.invoice_doc.is_return;
		},
		showChangeToggles() {
			return this.showCreditSaleToggle || (this.invoice_doc && this.invoice_doc.is_return);
		},
		canSetFullAmount() {
			if (!this.invoice_doc) {
				return false;
			}
			const total = this.invoiceTotal;
			return this.showPaymentMethodsSection && Math.abs(total) > 0;
		},
	},
	watch: {
		// Watch diff_payment to update paid_change
		diff_payment(newVal) {
			if (!this.is_user_editing_paid_change) {
				this.paid_change = -newVal;
			}
		},
		// Watch paid_change to validate and update credit_change
		paid_change(newVal) {
			const changeLimit = -this.diff_payment;
			if (newVal > changeLimit) {
				this.paid_change = changeLimit;
				this.credit_change = 0;
				this.paid_change_rules = ["Paid change can not be greater than total change!"];
			} else {
				this.paid_change_rules = [];
				this.credit_change = this.flt(newVal - changeLimit, this.currency_precision);
			}
		},
		// Watch loyalty_amount to handle loyalty points redemption
		loyalty_amount(value) {
			if (value > this.available_points_amount) {
				this.invoice_doc.loyalty_amount = 0;
				this.invoice_doc.redeem_loyalty_points = 0;
				this.invoice_doc.loyalty_points = 0;
				this.loyalty_amount = 0;
				this.eventBus.emit("show_message", {
					title: `Loyalty Amount can not be more than ${this.available_points_amount}`,
					color: "error",
				});
			} else {
				this.invoice_doc.loyalty_amount = this.flt(this.loyalty_amount);
				this.invoice_doc.redeem_loyalty_points = 1;
				this.invoice_doc.loyalty_points =
					this.flt(this.loyalty_amount) / this.customer_info.conversion_factor;
			}
		},
		// Watch redeemed_customer_credit to validate
		redeemed_customer_credit(newVal) {
			if (newVal > this.available_customer_credit) {
				this.redeemed_customer_credit = this.available_customer_credit;
				this.eventBus.emit("show_message", {
					title: `You can redeem customer credit up to ${this.available_customer_credit}`,
					color: "error",
				});
			}
		},
		redeem_customer_credit(value) {
			this.get_available_credit(value);
		},
		// Recalculate total redeemed credit whenever credit entries change
		customer_credit_dict: {
			handler(newVal) {
				const total = newVal.reduce((sum, row) => sum + this.flt(row.credit_to_redeem || 0), 0);
				this.redeemed_customer_credit = this.flt(total, this.currency_precision);
			},
			deep: true,
		},
		// Watch sales_person to update sales_team
		sales_person(newVal) {
			if (newVal) {
				this.invoice_doc.sales_team = [
					{
						sales_person: newVal,
						allocated_percentage: 100,
					},
				];
				console.log("Updated sales_team with sales_person:", newVal);
			} else {
				this.invoice_doc.sales_team = [];
				console.log("Cleared sales_team");
			}
		},
		// Watch is_credit_sale to reset cash payments
		is_credit_sale(newVal) {
			if (newVal) {
				// If credit sale is enabled, set cash payment to 0
				this.invoice_doc.payments.forEach((payment) => {
					if (payment.mode_of_payment.toLowerCase() === "cash") {
						payment.amount = 0;
					}
				});
			} else {
				// If credit sale is disabled, set cash payment to invoice total
				const total = this.invoiceTotal;
				this.invoice_doc.payments.forEach((payment) => {
					if (payment.mode_of_payment.toLowerCase() === "cash") {
						payment.amount = total;
					}
				});
			}
		},
		// Watch is_credit_return to toggle cashback payments
		is_credit_return(newVal) {
			if (newVal) {
				this.is_cashback = false;
				// Clear any payment amounts
				this.invoice_doc.payments.forEach((payment) => {
					payment.amount = 0;
					if (payment.base_amount !== undefined) {
						payment.base_amount = 0;
					}
				});
			} else {
				this.is_cashback = true;
				// Ensure default negative payment for returns
				this.ensureReturnPaymentsAreNegative();
			}
		},
	},
	methods: {
		openNumericKeypad({
			initialValue = 0,
			title = frappe._("Enter Amount"),
			helperText = "",
			allowDecimal = true,
			allowNegative = false,
			decimalPlaces = this.currency_precision,
			onConfirm = null,
		}) {
			this.numericKeypad.title = title;
			this.numericKeypad.helperText = helperText;
			this.numericKeypad.allowDecimal = allowDecimal;
			this.numericKeypad.allowNegative = allowNegative;
			this.numericKeypad.decimalPlaces = decimalPlaces ?? this.currency_precision;
			this.numericKeypad.apply = typeof onConfirm === "function" ? onConfirm : null;
			const preparedValue = this.prepareKeypadValue(initialValue, this.numericKeypad.decimalPlaces);
			this.numericKeypad.value = preparedValue;
			this.numericKeypad.visible = true;
		},
		closeNumericKeypad() {
			this.numericKeypad.visible = false;
			this.numericKeypad.apply = null;
		},
		handleNumericKeypadConfirm(rawValue) {
			const precision = this.numericKeypad.decimalPlaces ?? this.currency_precision;
			const numericValue = this.parseKeypadNumber(rawValue, precision);
			if (typeof this.numericKeypad.apply === "function") {
				this.numericKeypad.apply(numericValue);
			}
			this.closeNumericKeypad();
		},
		prepareKeypadValue(value, precision = this.currency_precision) {
			const parsed = this.parseKeypadNumber(value, precision);
			const factor = Number.isInteger(precision) ? precision : this.currency_precision;
			const normalized = factor > 0 ? Number(parsed.toFixed(factor)) : parsed;
			return String(normalized).replace(/^-0$/, "0");
		},
		parseKeypadNumber(value, precision = this.currency_precision) {
			const parsed = parseFloat(formatUtils.fromArabicNumerals(String(value || 0)));
			if (Number.isNaN(parsed)) {
				return 0;
			}
			return this.flt(parsed, precision);
		},
		openPaymentAmountKeypad(payment) {
			if (!payment) {
				return;
			}
			this.set_rest_amount(payment);
			const isReturn = this.invoice_doc?.is_return || this.invoiceType === "Return";
			const initialValue = Math.abs(parseFloat(payment.amount || 0)) || 0;
			const helperText = isReturn
				? frappe._("Return payments are stored as negative amounts automatically.")
				: "";
			this.openNumericKeypad({
				initialValue,
				title: frappe._("Payment Amount"),
				helperText,
				allowDecimal: true,
				allowNegative: isReturn,
				onConfirm: (value) => {
					let resolved = value;
					if (isReturn) {
						resolved = -Math.abs(resolved);
					}
					payment.amount = resolved;
					if (payment.base_amount !== undefined) {
						payment.base_amount = resolved;
					}
					this.normalizePaymentAmount(payment);
				},
			});
		},
		handlePaymentAmountFocus(payment) {
			this.set_rest_amount(payment);
			if (this.prefersTouchKeypad) {
				this.openPaymentAmountKeypad(payment);
			}
		},
		openPaidChangeKeypad() {
			this.openNumericKeypad({
				initialValue: Math.abs(this.paid_change || 0),
				title: frappe._("Paid Change"),
				helperText: "",
				onConfirm: (value) => {
					this.paid_change = value;
					this.onPaidChangeBlur();
				},
			});
		},
		openLoyaltyKeypad() {
			this.openNumericKeypad({
				initialValue: this.loyalty_amount || 0,
				title: frappe._("Redeem Loyalty Amount"),
				helperText: frappe._("Available: {0}", [this.formatCurrency(this.available_points_amount || 0)]),
				onConfirm: (value) => {
					this.loyalty_amount = value;
				},
			});
		},
		openCustomerCreditKeypad(credit) {
			if (!credit) {
				return;
			}
			this.openNumericKeypad({
				initialValue: credit.credit_to_redeem || 0,
				title: frappe._("Redeem Customer Credit"),
				helperText: frappe._("Available: {0}", [this.formatCurrency(credit.total_credit || 0)]),
				onConfirm: (value) => {
					credit.credit_to_redeem = value;
					this.normalizeCustomerCredit(credit);
				},
			});
		},
		handleInvoiceTotalsUpdate(updatedDoc) {
			if (!updatedDoc) {
				return;
			}
			if (!this.invoice_doc) {
				this.invoice_doc = updatedDoc;
				return;
			}
			const fieldsToSync = [
				"rounded_total",
				"grand_total",
				"total",
				"net_total",
				"total_taxes_and_charges",
				"base_total",
				"base_grand_total",
				"base_total_taxes_and_charges",
				"discount_amount",
				"currency",
				"conversion_rate",
				"outstanding_amount",
				"due_date",
				"in_words",
			];
			fieldsToSync.forEach((field) => {
				if (Object.prototype.hasOwnProperty.call(updatedDoc, field)) {
					this.$set(this.invoice_doc, field, updatedDoc[field]);
				}
			});
			if (Object.prototype.hasOwnProperty.call(updatedDoc, "taxes")) {
				this.$set(this.invoice_doc, "taxes", Array.isArray(updatedDoc.taxes) ? [...updatedDoc.taxes] : updatedDoc.taxes);
			}
			if (Object.prototype.hasOwnProperty.call(updatedDoc, "rounded_total_export")) {
				this.$set(this.invoice_doc, "rounded_total_export", updatedDoc.rounded_total_export);
			}
			this.$forceUpdate();
		},
		resolveInvoiceTotal(doc) {
			if (!doc) {
				return 0;
			}
			const parseAmount = (value) => {
				const parsed = parseFloat(formatUtils.fromArabicNumerals(String(value ?? 0)));
				return Number.isNaN(parsed) ? 0 : parsed;
			};
			const multiCurrency = Boolean(
				this.pos_profile?.posa_allow_multi_currency &&
				doc.currency &&
				this.pos_profile?.currency &&
				doc.currency !== this.pos_profile.currency,
			);
			if (multiCurrency) {
				return parseAmount(doc.grand_total);
			}
			if (!this.isRoundedTotalDisabled && doc.rounded_total != null) {
				return parseAmount(doc.rounded_total);
			}
			if (doc.grand_total != null) {
				return parseAmount(doc.grand_total);
			}
			if (doc.rounded_total != null) {
				return parseAmount(doc.rounded_total);
			}
			return 0;
		},
		// Go back to invoice view and reset customer readonly
		back_to_invoice() {
			this.showModal = false;
			this.eventBus.emit("show_payment", "false");
			this.eventBus.emit("set_customer_readonly", false);
			this.$nextTick(() => {
				this.eventBus.emit("focus_item_search");
			});
		},
		// Highlight and focus the submit button when payment screen opens
		handleShowPayment(data) {
			if (data === "true") {
				this.showModal = true;
				this.$nextTick(() => {
					const containerRef = this.$refs.paymentContainer;
					const containerEl = containerRef && containerRef.$el ? containerRef.$el : containerRef;
					if (containerEl) {
						if (typeof containerEl.scrollTo === "function") {
							containerEl.scrollTo({ top: 0, behavior: "auto" });
						} else if (Object.prototype.hasOwnProperty.call(containerEl, "scrollTop")) {
							containerEl.scrollTop = 0;
						}
					}
					setTimeout(() => {
						const btn = this.$refs.submitButton;
						const el = btn && btn.$el ? btn.$el : btn;
						if (el && typeof el.focus === "function") {
							try {
								el.focus({ preventScroll: true });
							} catch (err) {
								el.focus();
							}
						}
						if (el) {
							this.highlightSubmit = true;
						}
					}, 120);
				});
			} else {
				this.showModal = false;
				this.highlightSubmit = false;
			}
		},
		normalizePaymentAmount(payment) {
			if (!payment) {
				return;
			}
			const raw = formatUtils.fromArabicNumerals
				? formatUtils.fromArabicNumerals(String(payment.amount ?? 0))
				: payment.amount;
			const value = this.flt(raw || 0, this.currency_precision);
			payment.amount = value;
			if (payment.base_amount !== undefined) {
				payment.base_amount = value;
			}
			this.$forceUpdate();
		},
		clearPaymentAmount(payment) {
			if (!payment) {
				return;
			}
			payment.amount = 0;
			if (payment.base_amount !== undefined) {
				payment.base_amount = 0;
			}
			this.normalizePaymentAmount(payment);
		},
		openPhoneDialog(payment) {
			if (payment) {
				payment.amount = 0;
				if (payment.base_amount !== undefined) {
					payment.base_amount = 0;
				}
				this.normalizePaymentAmount(payment);
			}
			this.phone_dialog = true;
		},
		isMpesaPayment(payment) {
			if (!payment) {
				return false;
			}
			return (
				Array.isArray(this.mpesa_modes) &&
				this.mpesa_modes.includes(payment.mode_of_payment) &&
				payment.type === "Bank"
			);
		},
		onPaidChangeFocus() {
			this.is_user_editing_paid_change = true;
			if (this.prefersTouchKeypad) {
				this.openPaidChangeKeypad();
			}
		},
		onPaidChangeBlur() {
			const raw = formatUtils.fromArabicNumerals
				? formatUtils.fromArabicNumerals(String(this.paid_change ?? 0))
				: this.paid_change;
			const value = this.flt(raw || 0, this.currency_precision);
			this.paid_change = value;
			this.is_user_editing_paid_change = false;
			this.showPaidChange();
		},
		formatDateInput(value) {
			if (!value) {
				return "";
			}
			return this.formatDate(value) || "";
		},
		handleDeliveryDate(value) {
			this.new_delivery_date = value;
			this.update_delivery_date();
		},
		handlePODate(value) {
			this.new_po_date = value;
			this.update_po_date();
		},
		handleCreditDueDate(value) {
			this.new_credit_due_date = value;
			this.update_credit_due_date();
		},
		normalizeCustomerCredit(credit) {
			if (!credit) {
				return;
			}
			const raw = formatUtils.fromArabicNumerals
				? formatUtils.fromArabicNumerals(String(credit.credit_to_redeem ?? 0))
				: credit.credit_to_redeem;
			let value = this.flt(raw || 0, this.currency_precision);
			const max = this.flt(credit.total_credit || 0, this.currency_precision);
			if (value > max) {
				value = max;
			}
			if (value < 0) {
				value = 0;
			}
			credit.credit_to_redeem = value;
			this.$forceUpdate();
		},
		// Reset all cash payments to zero
		reset_cash_payments() {
			this.invoice_doc.payments.forEach((payment) => {
				if (payment.mode_of_payment.toLowerCase() === "cash") {
					payment.amount = 0;
				}
			});
		},
		// Ensure all payments are negative for return invoices
		ensureReturnPaymentsAreNegative() {
			if (!this.invoice_doc || !this.invoice_doc.is_return || !this.is_cashback) {
				return;
			}
			// Check if any payment amount is set
			let hasPaymentSet = false;
			this.invoice_doc.payments.forEach((payment) => {
				if (Math.abs(payment.amount) > 0) {
					hasPaymentSet = true;
				}
			});
			// If no payment set, set the default one
			if (!hasPaymentSet) {
				const default_payment = this.invoice_doc.payments.find((payment) => payment.default === 1);
				if (default_payment) {
					const amount = this.resolveInvoiceTotal(this.invoice_doc);
					default_payment.amount = -Math.abs(amount);
					if (default_payment.base_amount !== undefined) {
						default_payment.base_amount = -Math.abs(amount);
					}
				}
			}
			// Ensure all set payments are negative
			this.invoice_doc.payments.forEach((payment) => {
				if (payment.amount > 0) {
					payment.amount = -Math.abs(payment.amount);
				}
				if (payment.base_amount !== undefined && payment.base_amount > 0) {
					payment.base_amount = -Math.abs(payment.base_amount);
				}
			});
		},
		// Submit payment after validation
		async submit(event, payment_received = false, print = false) {
			// For return invoices, ensure payment amounts are negative
			if (this.invoice_doc.is_return) {
				this.ensureReturnPaymentsAreNegative();
			}
			const invoiceTotal = this.invoiceTotal;
			// Validate total payments only if not credit sale and invoice total is not zero
			if (
				!this.is_credit_sale &&
				!this.invoice_doc.is_return &&
				this.total_payments <= 0 &&
				invoiceTotal > 0
			) {
				this.eventBus.emit("show_message", {
					title: `Please enter payment amount`,
					color: "error",
				});
				frappe.utils.play_sound("error");
				return;
			}
			// Validate cash payments when credit sale is off
			if (!this.is_credit_sale && !this.invoice_doc.is_return) {
				let has_cash_payment = false;
				let cash_amount = 0;
				this.invoice_doc.payments.forEach((payment) => {
					if (payment.mode_of_payment.toLowerCase().includes("cash")) {
						has_cash_payment = true;
						cash_amount = this.flt(payment.amount);
					}
				});
				if (has_cash_payment && cash_amount > 0) {
					if (
						!this.pos_profile.posa_allow_partial_payment &&
						cash_amount < invoiceTotal &&
						invoiceTotal > 0
					) {
						this.eventBus.emit("show_message", {
							title: `Cash payment cannot be less than invoice total when partial payment is not allowed`,
							color: "error",
						});
						frappe.utils.play_sound("error");
						return;
					}
				}
			}
			// Validate partial payments only if not credit sale and invoice total is not zero
			if (
				!this.is_credit_sale &&
				!this.pos_profile.posa_allow_partial_payment &&
				this.total_payments < invoiceTotal &&
				invoiceTotal > 0
			) {
				this.eventBus.emit("show_message", {
					title: `The amount paid is not complete`,
					color: "error",
				});
				frappe.utils.play_sound("error");
				return;
			}
			// Validate phone payment
			let phone_payment_is_valid = true;
			if (!payment_received) {
				this.invoice_doc.payments.forEach((payment) => {
					if (payment.type === "Phone" && ![0, "0", "", null, undefined].includes(payment.amount)) {
						phone_payment_is_valid = false;
					}
				});
				if (!phone_payment_is_valid) {
					this.eventBus.emit("show_message", {
						title: __("Please request phone payment or use another payment method"),
						color: "error",
					});
					frappe.utils.play_sound("error");
					return;
				}
			}
			// Validate paid_change
			if (this.paid_change > -this.diff_payment) {
				this.eventBus.emit("show_message", {
					title: `Paid change cannot be greater than total change!`,
					color: "error",
				});
				frappe.utils.play_sound("error");
				return;
			}
			// Validate cashback
			let total_change = this.flt(this.flt(this.paid_change) + this.flt(-this.credit_change));
			if (this.is_cashback && total_change !== -this.diff_payment) {
				this.eventBus.emit("show_message", {
					title: `Error in change calculations!`,
					color: "error",
				});
				frappe.utils.play_sound("error");
				return;
			}
			// Validate customer credit redemption
			let credit_calc_check = this.customer_credit_dict.filter((row) => {
				return this.flt(row.credit_to_redeem) > this.flt(row.total_credit);
			});
			if (credit_calc_check.length > 0) {
				this.eventBus.emit("show_message", {
					title: `Redeemed credit cannot be greater than its total.`,
					color: "error",
				});
				frappe.utils.play_sound("error");
				return;
			}
			if (
				!this.invoice_doc.is_return &&
				this.redeemed_customer_credit > invoiceTotal
			) {
				this.eventBus.emit("show_message", {
					title: `Cannot redeem customer credit more than invoice total`,
					color: "error",
				});
				frappe.utils.play_sound("error");
				return;
			}
			// Validate stock availability before submitting
			if (!isOffline()) {
				try {
					const itemsToCheck = this.invoice_doc.items.filter((it) => !it.is_bundle);
					const stockCheck = await frappe.call({
						method: "posawesome.posawesome.api.invoices.validate_cart_items",
						args: { items: JSON.stringify(itemsToCheck) },
					});
					if (stockCheck.message && stockCheck.message.length) {
						const msg = stockCheck.message
							.map(
								(e) =>
									`${e.item_code} (${e.warehouse}) - ${this.formatFloat(e.available_qty)}`,
							)
							.join("\n");
						const blocking =
							!this.stock_settings.allow_negative_stock || this.blockSaleBeyondAvailableQty;
						this.eventBus.emit("show_message", {
							title: blocking
								? __("Insufficient stock:\n{0}", [msg])
								: __("Stock is lower than requested:\n{0}", [msg]),
							color: blocking ? "error" : "warning",
						});
						if (blocking) {
							frappe.utils.play_sound("error");
							this.loading = false;
							return;
						}
					}
				} catch (e) {
					console.error("Stock validation failed", e);
				}
			}

			// Proceed to submit the invoice
			this.loading = true;
			this.submit_invoice(print);
		},
		// Submit invoice to backend after all validations
		submit_invoice(print) {
			// For return invoices, ensure payments are negative one last time
			if (this.invoice_doc.is_return) {
				this.ensureReturnPaymentsAreNegative();
			}
			let totalPayedAmount = 0;
			this.invoice_doc.payments.forEach((payment) => {
				payment.amount = this.flt(payment.amount);
				totalPayedAmount += payment.amount;
			});
			if (this.invoice_doc.is_return && totalPayedAmount === 0) {
				this.invoice_doc.is_pos = 0;
			}
			if (this.customer_credit_dict.length) {
				this.customer_credit_dict.forEach((row) => {
					row.credit_to_redeem = this.flt(row.credit_to_redeem);
				});
			}
			let data = {
				total_change: !this.invoice_doc.is_return ? -this.diff_payment : 0,
				paid_change: !this.invoice_doc.is_return ? this.paid_change : 0,
				credit_change: -this.credit_change,
				redeemed_customer_credit: this.redeemed_customer_credit,
				customer_credit_dict: this.customer_credit_dict,
				is_cashback: this.is_cashback,
			};
			const vm = this;

			if (isOffline()) {
				try {
					saveOfflineInvoice({ data: data, invoice: this.invoice_doc });
					this.eventBus.emit("pending_invoices_changed", getPendingOfflineInvoiceCount());
					vm.eventBus.emit("show_message", {
						title: __("Invoice saved offline"),
						color: "warning",
					});
					if (print) {
						this.print_offline_invoice(this.invoice_doc);
					}
					vm.eventBus.emit("clear_invoice");
					vm.eventBus.emit("focus_item_search");
					vm.eventBus.emit("reset_posting_date");
					vm.back_to_invoice();
					vm.loading = false;
					return;
				} catch (error) {
					vm.eventBus.emit("show_message", {
						title: __("Cannot Save Offline Invoice: ") + (error.message || __("Unknown error")),
						color: "error",
					});
					vm.loading = false;
					return;
				}
			}
			frappe.call({
				method:
					this.invoiceType === "Order" && this.pos_profile.posa_create_only_sales_order
						? "posawesome.posawesome.api.sales_orders.submit_sales_order"
						: this.invoiceType === "Quotation"
							? "posawesome.posawesome.api.quotations.submit_quotation"
							: "posawesome.posawesome.api.invoices.submit_invoice",
				args: {
					data: data,
					invoice: this.invoice_doc,
					order: this.invoice_doc,
				},
				callback: function (r) {
					if (r.exc) {
						console.error("Error submitting invoice:", r.exc);
						// Show detailed error message to help debugging
						let errorMsg = r.exc.toString();
						if (errorMsg.includes("Amount must be negative")) {
							vm.eventBus.emit("show_message", {
								title: __("Fixing payment amounts for return invoice..."),
								color: "warning",
							});
							// Force fix the amounts
							vm.invoice_doc.payments.forEach((payment) => {
								if (payment.amount > 0) {
									payment.amount = -Math.abs(payment.amount);
								}
								if (payment.base_amount > 0) {
									payment.base_amount = -Math.abs(payment.base_amount);
								}
							});
							// Retry submission once
							console.log("Retrying submission with fixed payment amounts");
							setTimeout(() => {
								vm.submit_invoice(print);
							}, 500);
						} else {
							vm.eventBus.emit("show_message", {
								title: __("Error submitting invoice: ") + errorMsg,
								color: "error",
							});
						}
						vm.loading = false;
						return;
					}
					if (!r.message) {
						vm.eventBus.emit("show_message", {
							title: __("Error submitting invoice: No response from server"),
							color: "error",
						});
						vm.loading = false;
						return;
					}
					if (print) {
						vm.load_print_page();
					}
					vm.customer_credit_dict = [];
					vm.redeem_customer_credit = false;
					vm.is_cashback = true;
					vm.is_credit_return = false;
					vm.sales_person = "";
					vm.eventBus.emit("set_last_invoice", vm.invoice_doc.name);
					vm.eventBus.emit("show_message", {
						title:
							vm.invoiceType === "Order" && vm.pos_profile.posa_create_only_sales_order
								? __("Sales Order {0} is Submitted", [r.message.name])
								: vm.invoiceType === "Quotation"
									? __("Quotation {0} is Submitted", [r.message.name])
									: __("Invoice {0} is Submitted", [r.message.name]),
						color: "success",
					});
					frappe.utils.play_sound("submit");
					// Update local stock quantities immediately after successful
					// invoice submission so item availability reflects changes
					updateLocalStock(vm.invoice_doc.items || []);
					vm.addresses = [];
					vm.eventBus.emit("clear_invoice");
					vm.eventBus.emit("focus_item_search");
					vm.eventBus.emit("reset_posting_date");
					vm.back_to_invoice();
					vm.loading = false;
				},
			});
		},
		// Set full amount for a payment method (or negative for returns)
		set_full_amount(payment) {
			if (!this.invoice_doc || !Array.isArray(this.invoice_doc.payments) || !payment) {
				return;
			}
		const isReturn = this.invoice_doc.is_return || this.invoiceType === "Return";
		const totalAmount = this.invoiceTotal;
			if (!totalAmount) {
				return;
			}

			this.invoice_doc.payments.forEach((row) => {
				row.amount = 0;
				if (row.base_amount !== undefined) {
					row.base_amount = 0;
				}
				this.normalizePaymentAmount(row);
			});

			const targetPayment = this.invoice_doc.payments.find(
				(row) => row === payment || (row.idx === payment.idx && row.mode_of_payment === payment.mode_of_payment),
			);
			if (!targetPayment) {
				return;
			}

		const amount = isReturn ? -Math.abs(totalAmount) : totalAmount;
			targetPayment.amount = amount;
			if (targetPayment.base_amount !== undefined) {
				targetPayment.base_amount = isReturn ? -Math.abs(amount) : amount;
			}
			this.normalizePaymentAmount(targetPayment);
		},
		// Set remaining amount for a payment method when focused
		set_rest_amount(payment) {
			if (!this.invoice_doc || !Array.isArray(this.invoice_doc.payments) || !payment) {
				return;
			}
			const isReturn = this.invoice_doc.is_return || this.invoiceType === "Return";
			const targetPayment = this.invoice_doc.payments.find(
				(row) => row === payment || (row.idx === payment.idx && row.mode_of_payment === payment.mode_of_payment),
			);
			if (!targetPayment) {
				return;
			}
			if (Math.abs(targetPayment.amount || 0) > 0) {
				return;
			}
			if (!isReturn && this.diff_payment <= 0) {
				return;
			}
			let amount = this.diff_payment;
			if (isReturn) {
				amount = -Math.abs(amount);
			}
			if (!isReturn && amount <= 0) {
				return;
			}
			targetPayment.amount = amount;
			if (targetPayment.base_amount !== undefined) {
				targetPayment.base_amount = isReturn ? -Math.abs(amount) : amount;
			}
			this.normalizePaymentAmount(targetPayment);
		},
		// Clear all payment amounts
		clear_all_amounts() {
			if (!this.invoice_doc || !Array.isArray(this.invoice_doc.payments)) {
				return;
			}
			this.invoice_doc.payments.forEach((payment) => {
				payment.amount = 0;
				if (payment.base_amount !== undefined) {
					payment.base_amount = 0;
				}
				this.normalizePaymentAmount(payment);
			});
		},
		// Open print page for invoice
		load_print_page() {
			const print_format = this.pos_profile.print_format_for_online || this.pos_profile.print_format;
			const letter_head = this.pos_profile.letter_head || 0;
			const doctype = this.pos_profile.create_pos_invoice_instead_of_sales_invoice
				? "POS Invoice"
				: "Sales Invoice";
			const url =
				frappe.urllib.get_base_url() +
				"/printview?doctype=" +
				encodeURIComponent(doctype) +
				"&name=" +
				this.invoice_doc.name +
				"&trigger_print=1" +
				"&format=" +
				print_format +
				"&no_letterhead=" +
				letter_head;
			if (this.pos_profile.posa_silent_print) {
				silentPrint(url);
			} else {
				const printWindow = window.open(url, "Print");
				printWindow.addEventListener(
					"load",
					function () {
						printWindow.print();
					},
					{ once: true },
				);
			}
		},
		// Print invoice using a more detailed offline template
		async print_offline_invoice(invoice) {
			if (!invoice) return;
			const html = await renderOfflineInvoiceHTML(invoice);
			const win = window.open("", "_blank");
			win.document.write(html);
			win.document.close();
			win.focus();
			win.print();
		},
		// Validate due date (should not be in the past)
		validate_due_date() {
			const today = frappe.datetime.now_date();
			const new_date = Date.parse(this.invoice_doc.due_date);
			const parse_today = Date.parse(today);
			if (new_date < parse_today) {
				this.invoice_doc.due_date = today;
			}
		},
		// Keyboard shortcut for payment submit (Ctrl+X)
		shortPay(e) {
			if (e.key.toLowerCase() === "x" && (e.ctrlKey || e.metaKey)) {
				e.preventDefault();
				e.stopPropagation();
				if (this.invoice_doc && this.invoice_doc.payments) {
					this.submit_invoice();
				}
			}
		},
		// Get available customer credit and auto-allocate
		get_available_credit(use_credit) {
			this.clear_all_amounts();
			if (use_credit) {
				frappe
					.call("posawesome.posawesome.api.payments.get_available_credit", {
						customer: this.invoice_doc.customer,
						company: this.pos_profile.company,
					})
					.then((r) => {
						const data = r.message;
						if (data.length) {
							const amount = this.invoiceTotal;
							let remainAmount = amount;
							data.forEach((row) => {
								if (remainAmount > 0) {
									if (remainAmount >= row.total_credit) {
										row.credit_to_redeem = row.total_credit;
										remainAmount -= row.total_credit;
									} else {
										row.credit_to_redeem = remainAmount;
										remainAmount = 0;
									}
								} else {
									row.credit_to_redeem = 0;
								}
							});
							this.customer_credit_dict = data;
						} else {
							this.customer_credit_dict = [];
						}
					});
			} else {
				this.customer_credit_dict = [];
			}
		},
		// Get customer addresses for shipping
		get_addresses() {
			const vm = this;
			if (!vm.invoice_doc || !vm.invoice_doc.customer) {
				vm.addresses = [];
				return;
			}
			frappe.call({
				method: "posawesome.posawesome.api.customers.get_customer_addresses",
				args: { customer: vm.invoice_doc.customer },
				async: true,
				callback: function (r) {
					if (!r.exc) {
						vm.addresses = r.message;
					} else {
						vm.addresses = [];
					}
				},
			});
		},
		// Filter addresses for autocomplete
		addressFilter(item, queryText) {
			const searchText = queryText.toLowerCase();
			return (
				(item.address_title && item.address_title.toLowerCase().includes(searchText)) ||
				(item.address_line1 && item.address_line1.toLowerCase().includes(searchText)) ||
				(item.address_line2 && item.address_line2.toLowerCase().includes(searchText)) ||
				(item.city && item.city.toLowerCase().includes(searchText)) ||
				(item.name && item.name.toLowerCase().includes(searchText))
			);
		},
		// Open dialog to add new address
		new_address() {
			if (!this.invoice_doc || !this.invoice_doc.customer) {
				this.eventBus.emit("show_message", {
					title: __("Please select a customer first"),
					color: "error",
				});
				return;
			}
			this.eventBus.emit("open_new_address", this.invoice_doc.customer);
		},
		// Get sales person names from API/localStorage
		get_sales_person_names() {
			const vm = this;
			if (vm.pos_profile.posa_local_storage && getSalesPersonsStorage().length) {
				try {
					vm.sales_persons = getSalesPersonsStorage();
				} catch (e) {
					console.error(e);
				}
			}
			frappe.call({
				method: "posawesome.posawesome.api.utilities.get_sales_person_names",
				callback: function (r) {
					if (r.message && r.message.length > 0) {
						vm.sales_persons = r.message.map((sp) => ({
							value: sp.name,
							title: sp.sales_person_name,
							sales_person_name: sp.sales_person_name,
							name: sp.name,
						}));
						if (vm.pos_profile.posa_local_storage) {
							setSalesPersonsStorage(vm.sales_persons);
						}
					} else {
						vm.sales_persons = [];
					}
				},
			});
		},
		// Request payment for phone type
		request_payment() {
			this.phone_dialog = false;
			const vm = this;
			if (!this.invoice_doc.contact_mobile) {
				this.eventBus.emit("show_message", {
					title: __("Please set the customer's mobile number"),
					color: "error",
				});
				this.eventBus.emit("open_edit_customer");
				this.back_to_invoice();
				return;
			}
			this.eventBus.emit("freeze", { title: __("Waiting for payment...") });
			this.invoice_doc.payments.forEach((payment) => {
				payment.amount = this.flt(payment.amount);
			});
			let formData = { ...this.invoice_doc };
			formData["total_change"] = !this.invoice_doc.is_return ? -this.diff_payment : 0;
			formData["paid_change"] = !this.invoice_doc.is_return ? this.paid_change : 0;
			formData["credit_change"] = -this.credit_change;
			formData["redeemed_customer_credit"] = this.redeemed_customer_credit;
			formData["customer_credit_dict"] = this.customer_credit_dict;
			formData["is_cashback"] = this.is_cashback;
			frappe
				.call({
					method: "posawesome.posawesome.api.invoices.update_invoice",
					args: { data: formData },
					async: false,
					callback: function (r) {
						if (r.message) {
							vm.invoice_doc = r.message;
						}
					},
				})
				.then(() => {
					frappe
						.call({
							method: "posawesome.posawesome.api.payments.create_payment_request",
							args: { doc: vm.invoice_doc },
						})
						.fail(() => {
							vm.eventBus.emit("unfreeze");
							vm.eventBus.emit("show_message", {
								title: __("Payment request failed"),
								color: "error",
							});
						})
						.then(({ message }) => {
							const payment_request_name = message.name;
							setTimeout(() => {
								frappe.db
									.get_value("Payment Request", payment_request_name, [
										"status",
										"grand_total",
									])
									.then(({ message }) => {
										if (message.status !== "Paid") {
											vm.eventBus.emit("unfreeze");
											vm.eventBus.emit("show_message", {
												title: __(
													"Payment Request took too long to respond. Please try requesting for payment again",
												),
												color: "error",
											});
										} else {
											vm.eventBus.emit("unfreeze");
											vm.eventBus.emit("show_message", {
												title: __("Payment of {0} received successfully.", [
													vm.formatCurrency(
														message.grand_total,
														vm.invoice_doc.currency,
														0,
													),
												]),
												color: "success",
											});
											frappe.db
												.get_doc(vm.invoice_doc.doctype, vm.invoice_doc.name)
												.then((doc) => {
													vm.invoice_doc = doc;
													vm.submit(null, true);
												});
										}
									});
							}, 30000);
						});
				});
		},
		// Get M-Pesa payment modes from backend
		get_mpesa_modes() {
			const vm = this;
			frappe.call({
				method: "posawesome.posawesome.api.m_pesa.get_mpesa_mode_of_payment",
				args: { company: vm.pos_profile.company },
				async: true,
				callback: function (r) {
					if (!r.exc) {
						vm.mpesa_modes = r.message;
					} else {
						vm.mpesa_modes = [];
					}
				},
			});
		},
		// Check if payment is M-Pesa C2B
		is_mpesa_c2b_payment(payment) {
			if (this.mpesa_modes.includes(payment.mode_of_payment) && payment.type === "Bank") {
				payment.amount = 0;
				return true;
			} else {
				return false;
			}
		},
		// Open M-Pesa payment dialog
		mpesa_c2b_dialog(payment) {
			const data = {
				company: this.pos_profile.company,
				mode_of_payment: payment.mode_of_payment,
				customer: this.invoice_doc.customer,
			};
			this.eventBus.emit("open_mpesa_payments", data);
		},
		// Set M-Pesa payment as customer credit
		set_mpesa_payment(payment) {
			this.pos_profile.use_customer_credit = true;
			this.redeem_customer_credit = true;
		const invoiceAmount = this.invoiceTotal;
			let amount =
				payment.unallocated_amount > invoiceAmount ? invoiceAmount : payment.unallocated_amount;
			amount = amount > 0 ? amount : 0;
			const advance = {
				type: "Advance",
				credit_origin: payment.name,
				total_credit: this.flt(payment.unallocated_amount),
				credit_to_redeem: this.flt(amount),
			};
			this.clear_all_amounts();
			this.customer_credit_dict.push(advance);
		},
		// Update delivery date after selection
		update_delivery_date() {
			this.invoice_doc.posa_delivery_date = this.formatDate(this.new_delivery_date);
			// After setting delivery date, fetch addresses if not already loaded
			if (this.invoice_doc.customer && (!this.addresses || this.addresses.length === 0)) {
				this.get_addresses();
			}
		},
		// Update purchase order date after selection
		update_po_date() {
			this.invoice_doc.po_date = this.formatDate(this.new_po_date);
		},
		// Update credit due date after selection
		update_credit_due_date() {
			this.invoice_doc.due_date = this.formatDate(this.new_credit_due_date);
		},
		// Apply preset or typed number of days to set due date
		applyDuePreset(days) {
			if (days === null || days === "") {
				return;
			}
			const westernDays = formatUtils.fromArabicNumerals(String(days));
			if (isNaN(westernDays)) {
				return;
			}
			const parsed = parseInt(westernDays, 10);
			const d = new Date();
			d.setDate(d.getDate() + parsed);
			this.new_credit_due_date = this.formatDateDisplay(d);
			this.credit_due_days = parsed;
			this.update_credit_due_date();
		},
		// Apply days entered in dialog
		applyCustomDays() {
			this.applyDuePreset(this.custom_days_value);
			this.custom_days_dialog = false;
		},
		// Format date to YYYY-MM-DD
		formatDate(date) {
			if (!date) return null;
			if (typeof date === "string") {
				const western = formatUtils.fromArabicNumerals(date);
				if (/^\d{4}-\d{2}-\d{2}$/.test(western)) {
					return western;
				}
				if (/^\d{1,2}-\d{1,2}-\d{4}$/.test(western)) {
					const [d, m, y] = western.split("-");
					return `${y}-${m.padStart(2, "0")}-${d.padStart(2, "0")}`;
				}
				date = western;
			}
			const d = new Date(formatUtils.fromArabicNumerals(String(date)));
			if (!isNaN(d.getTime())) {
				const year = d.getFullYear();
				const month = `0${d.getMonth() + 1}`.slice(-2);
				const day = `0${d.getDate()}`.slice(-2);
				return `${year}-${month}-${day}`;
			}
			return formatUtils.fromArabicNumerals(String(date));
		},

		formatDateDisplay(date) {
			if (!date) return "";
			const western = formatUtils.fromArabicNumerals(String(date));
			if (typeof date === "string" && /^\d{4}-\d{2}-\d{2}$/.test(western)) {
				const [y, m, d] = western.split("-");
				return formatUtils.toArabicNumerals(`${d}-${m}-${y}`);
			}
			const d = new Date(western);
			if (!isNaN(d.getTime())) {
				const year = d.getFullYear();
				const month = `0${d.getMonth() + 1}`.slice(-2);
				const day = `0${d.getDate()}`.slice(-2);
				return formatUtils.toArabicNumerals(`${day}-${month}-${year}`);
			}
			return formatUtils.toArabicNumerals(western);
		},
		// Show paid amount info message
		showPaidAmount() {
			this.eventBus.emit("show_message", {
				title: `Total Paid Amount: ${this.formatCurrency(this.total_payments)}`,
				color: "info",
			});
		},
		// Show diff payment info message
		showDiffPayment() {
			if (!this.invoice_doc) return;
			this.eventBus.emit("show_message", {
				title: `To Be Paid: ${this.formatCurrency(this.diff_payment)}`,
				color: "info",
			});
		},
		// Show paid change info message
		showPaidChange() {
			this.eventBus.emit("show_message", {
				title: `Paid Change: ${this.formatCurrency(this.paid_change)}`,
				color: "info",
			});
		},
		// Show credit change info message
		showCreditChange(value) {
			if (value > 0) {
				this.credit_change = value;
				this.paid_change = -this.diff_payment;
			} else {
				this.credit_change = 0;
			}
		},
		// Format currency value
		formatCurrency(value) {
			return this.$options.mixins[0].methods.formatCurrency.call(this, value, this.currency_precision);
		},
		// Get change amount for display
		get_change_amount() {
			return Math.max(0, this.total_payments - this.invoiceTotal);
		},
		// Sync any invoices stored offline and show pending/synced counts
		async syncPendingInvoices() {
			const pending = getPendingOfflineInvoiceCount();
			if (pending) {
				this.eventBus.emit("show_message", {
					title: `${pending} invoice${pending > 1 ? "s" : ""} pending for sync`,
					color: "warning",
				});
				this.eventBus.emit("pending_invoices_changed", pending);
			}
			if (isOffline()) {
				// Don't attempt to sync while offline; just update the counter
				return;
			}
			const result = await syncOfflineInvoices();
			if (result && (result.synced || result.drafted)) {
				if (result.synced) {
					this.eventBus.emit("show_message", {
						title: `${result.synced} offline invoice${result.synced > 1 ? "s" : ""} synced`,
						color: "success",
					});
				}
				if (result.drafted) {
					this.eventBus.emit("show_message", {
						title: `${result.drafted} offline invoice${result.drafted > 1 ? "s" : ""} saved as draft`,
						color: "warning",
					});
				}
			}
			this.eventBus.emit("pending_invoices_changed", getPendingOfflineInvoiceCount());
		},
	},
	// Lifecycle hook: created
	created() {
		// Register keyboard shortcut for payment
		document.addEventListener("keydown", this.shortPay.bind(this));
		this.syncPendingInvoices();
		this.eventBus.on("network-online", this.syncPendingInvoices);
		// Also sync when the server connection is re-established
		this.eventBus.on("server-online", this.syncPendingInvoices);
	},
	// Lifecycle hook: mounted
	mounted() {
		if (typeof window !== "undefined") {
			try {
				const coarseMatch = window.matchMedia?.("(pointer: coarse)")?.matches;
				this.prefersTouchKeypad = Boolean(coarseMatch || "ontouchstart" in window);
			} catch (err) {
				this.prefersTouchKeypad = false;
			}
		}
		this.$nextTick(() => {
			// Listen to various event bus events for POS actions
			this.eventBus.on("send_invoice_doc_payment", (invoice_doc) => {
				this.invoice_doc = invoice_doc;
				const default_payment = this.invoice_doc.payments.find((payment) => payment.default === 1);
				this.is_credit_sale = false;
				this.is_write_off_change = false;
				if (invoice_doc.is_return) {
					this.is_return = true;
					this.is_credit_return = false;
					// Reset all payment amounts to zero for returns
					invoice_doc.payments.forEach((payment) => {
						payment.amount = 0;
						payment.base_amount = 0;
					});
					// Set default payment to negative amount for returns
		if (default_payment) {
			const amount = this.resolveInvoiceTotal(invoice_doc);
			default_payment.amount = -Math.abs(amount);
			if (default_payment.base_amount !== undefined) {
				default_payment.base_amount = -Math.abs(amount);
			}
		}
		} else if (default_payment) {
			// For regular invoices, set positive amount
			const total = this.resolveInvoiceTotal(invoice_doc);
			default_payment.amount = this.flt(total, this.currency_precision);
			this.is_credit_return = false;
		}
				this.loyalty_amount = 0;
				this.redeemed_customer_credit = 0;
				// Only get addresses if customer exists
				if (invoice_doc.customer) {
					this.get_addresses();
				}
				this.get_sales_person_names();
			});
			this.eventBus.on("register_pos_profile", (data) => {
				this.pos_profile = data.pos_profile;
				this.stock_settings = data.stock_settings || {};
				this.get_mpesa_modes();
			});
			this.eventBus.on("add_the_new_address", (data) => {
				this.addresses.push(data);
				this.$forceUpdate();
			});
			this.eventBus.on("update_invoice_type", (data) => {
				this.invoiceType = data;
				if (this.invoice_doc && data !== "Order") {
					this.invoice_doc.posa_delivery_date = null;
					this.invoice_doc.posa_notes = null;
					this.invoice_doc.shipping_address_name = null;
				} else if (this.invoice_doc && data === "Order") {
					// Initialize delivery date to today when switching to Order type
					this.new_delivery_date = this.formatDateDisplay(frappe.datetime.now_date());
					this.update_delivery_date();
				}
				// Handle return invoices properly
				if (this.invoice_doc && data === "Return") {
					this.invoice_doc.is_return = 1;
					// Ensure payments are negative for returns
					this.ensureReturnPaymentsAreNegative();
					this.is_credit_return = false;
				}
			});
			this.eventBus.on("update_customer", (customer) => {
				if (this.customer !== customer) {
					this.customer_credit_dict = [];
					this.redeem_customer_credit = false;
					this.is_cashback = true;
					this.is_credit_return = false;
				}
			});
			this.eventBus.on("set_pos_settings", (data) => {
				this.pos_settings = data;
			});
			this.eventBus.on("set_customer_info_to_edit", (data) => {
				this.customer_info = data;
			});
			this.eventBus.on("set_mpesa_payment", (data) => {
				this.set_mpesa_payment(data);
			});
			this.eventBus.on("refresh_invoice_totals", this.handleInvoiceTotalsUpdate);
			// Clear any stored invoice when parent emits clear_invoice
			this.eventBus.on("clear_invoice", () => {
				this.invoice_doc = "";
				this.is_return = false;
				this.is_credit_return = false;
			});
			// Scroll to top when payment view is shown
			this.eventBus.on("show_payment", this.handleShowPayment);
		});
	},
	// Lifecycle hook: beforeUnmount
	beforeUnmount() {
		// Remove all event listeners
		this.eventBus.off("send_invoice_doc_payment");
		this.eventBus.off("register_pos_profile");
		this.eventBus.off("add_the_new_address");
		this.eventBus.off("update_invoice_type");
		this.eventBus.off("update_customer");
		this.eventBus.off("set_pos_settings");
		this.eventBus.off("set_customer_info_to_edit");
		this.eventBus.off("set_mpesa_payment");
		this.eventBus.off("clear_invoice");
		this.eventBus.off("network-online", this.syncPendingInvoices);
		this.eventBus.off("server-online", this.syncPendingInvoices);
		this.eventBus.off("show_payment", this.handleShowPayment);
		this.eventBus.off("refresh_invoice_totals", this.handleInvoiceTotalsUpdate);
	},
	// Lifecycle hook: unmounted
	unmounted() {
		// Remove keyboard shortcut listener
		document.removeEventListener("keydown", this.shortPay);
	},
};
</script>

<style scoped>
/* Modal-specific styles */
.payments-modal {
	height: 95vh;
	max-height: 95vh;
	display: flex;
	flex-direction: column;
}

.payments-modal .v-card-text {
	flex: 1;
	overflow: hidden;
}

.payments-modal .selection {
	height: calc(95vh - 120px);
	max-height: calc(95vh - 120px);
	border: none;
	box-shadow: none;
}

.payments-card {
	display: flex;
	flex-direction: column;
	padding: 14px;
	gap: 14px;
}

.payments-body {
	flex: 1 1 auto;
	overflow-y: auto;
	display: flex;
	flex-direction: column;
	gap: 12px;
	padding: 12px 8px 140px;
}

.payment-layout {
	display: flex;
	flex-direction: column;
	gap: 14px;
}

.payment-section {
	background-color: rgb(var(--v-theme-surface));
	border: 1px solid rgba(var(--v-theme-outline), 0.12);
	border-radius: 16px;
	padding: 14px 16px;
	box-shadow: 0 12px 30px rgba(15, 23, 42, 0.08);
	transition: box-shadow 0.2s ease, border-color 0.2s ease;
}

.payment-section:hover {
	box-shadow: 0 16px 36px rgba(15, 23, 42, 0.12);
	border-color: rgba(var(--v-theme-primary), 0.18);
}

.payment-section__header {
	display: flex;
	align-items: center;
	gap: 12px;
	margin-bottom: 8px;
}

.payment-section__icon {
	color: rgb(var(--v-theme-primary));
}

.payment-section__title {
	font-weight: 600;
	letter-spacing: 0.3px;
}

.payment-section__content {
	margin: 0;
	row-gap: 12px;
	column-gap: 12px;
}

.payment-section--summary {
	border-color: rgba(var(--v-theme-outline), 0.2) !important;
}

.payment-method-list {
	display: flex;
	flex-direction: column;
	gap: 16px;
}

.payment-method-card {
	background-color: rgb(var(--v-theme-surface));
	border: 1px solid rgba(var(--v-theme-outline), 0.12);
	border-radius: 14px;
	padding: 16px;
	display: flex;
	flex-direction: column;
	gap: 12px;
}

.payment-method-card__header {
	display: flex;
	flex-wrap: wrap;
	justify-content: space-between;
	align-items: center;
	gap: 8px;
}

.payment-method-card__title {
	display: flex;
	align-items: center;
	gap: 8px;
	font-weight: 600;
}

.payment-method-card__meta {
	font-size: 0.85rem;
	color: rgba(var(--v-theme-on-surface), 0.6);
}

.payment-method-card__actions {
	display: flex;
	flex-wrap: wrap;
	gap: 8px;
	align-items: center;
}

.payment-method-card__actions .v-btn {
	border-radius: 999px;
}

.payment-toggle-row {
	display: flex;
	flex-wrap: wrap;
	gap: 16px;
	margin-top: 12px;
}

.payment-toggle-row__switch {
	margin-right: 8px;
}

.payment-credit-toggle {
	margin-top: 8px;
}

.customer-credit-list {
	display: grid;
	gap: 12px;
	grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
}

.customer-credit-card {
	background-color: rgba(var(--v-theme-primary), 0.04);
	border: 1px solid rgba(var(--v-theme-primary), 0.12);
	border-radius: 12px;
	padding: 12px 14px;
	display: flex;
	flex-direction: column;
	gap: 10px;
}

.customer-credit-card__header {
	display: flex;
	justify-content: space-between;
	align-items: center;
	gap: 8px;
	font-weight: 600;
	color: rgba(var(--v-theme-on-surface), 0.82);
}

.customer-credit-card__header span {
	font-size: 0.85rem;
	font-weight: 500;
	color: rgba(var(--v-theme-on-surface), 0.6);
}

.customer-credit-empty {
	padding: 12px 0;
	font-size: 0.9rem;
	color: rgba(var(--v-theme-on-surface), 0.55);
}

.credit-presets {
	margin-top: 8px;
	display: flex;
	flex-wrap: wrap;
	gap: 8px;
}

.credit-presets .v-chip {
	font-size: 0.75rem;
	text-transform: uppercase;
}

.payment-empty-state {
	display: flex;
	flex-direction: column;
	align-items: center;
	justify-content: center;
	text-align: center;
	padding: 60px 16px;
	color: rgba(var(--v-theme-on-surface), 0.6);
}

.payments-actions {
	position: sticky;
	bottom: 0;
	background: rgb(var(--v-theme-surface)) !important;
	padding: 20px 24px;
	box-shadow: 0 -6px 18px rgba(15, 23, 42, 0.08);
	border-radius: 24px 24px 0 0;
	border: 1px solid rgba(var(--v-theme-outline), 0.16);
	z-index: 5;
}

.payments-actions-row {
	row-gap: 14px;
	column-gap: 16px;
}

.payments-actions-row>.v-col {
	display: flex;
}

.payments-actions-btn {
	width: 100%;
	height: 100%;
	border-radius: 18px !important;
	font-size: 1.05rem !important;
	font-weight: 600 !important;
}

.read-only-field :deep(.v-field) {
	background-color: rgba(var(--v-theme-surface), 0.6);
	border-color: rgba(var(--v-theme-outline), 0.2);
}

.read-only-field :deep(.v-field__input) {
	color: rgba(var(--v-theme-on-surface), 0.7);
	font-weight: 600;
}

.editable-field :deep(.v-field) {
	background-color: rgba(var(--v-theme-primary), 0.04);
	border-color: rgba(var(--v-theme-primary), 0.25);
}

.editable-field :deep(.v-field__outline) {
	opacity: 0.6;
}

.editable-field :deep(.v-field__input) {
	font-weight: 600;
}


.payment-action-btn {
	color: #fff !important;
}

.payment-action-btn :deep(.v-btn__content) {
	color: inherit !important;
}

.payment-action-btn:hover,
.payment-action-btn:focus-visible,
.payment-action-btn:active {
	color: #fff !important;
	box-shadow: 0 4px 12px rgba(0, 0, 0, 0.25) !important;
}

.payment-action-btn::before,
.payment-action-btn:hover::before,
.payment-action-btn:focus::before,
.payment-action-btn:focus-visible::before,
.payment-action-btn:active::before {
	opacity: 0 !important;
}


@media (max-width: 768px) {
	.payments-modal .selection {
		height: calc(100vh - 80px);
		max-height: calc(100vh - 80px);
	}

	.payments-card {
		padding: 12px;
		gap: 12px;
	}

	.payments-body {
		padding: 12px 4px 120px;
		gap: 20px;
	}

	.payment-section {
		padding: 16px;
		gap: 12px;
	}

	.payment-method-card {
		padding: 14px;
	}

	.payment-method-card__actions {
		gap: 6px;
	}

	.payment-toggle-row {
		gap: 12px;
	}

	.payments-actions {
		padding: 16px 18px;
		border-radius: 20px 20px 0 0;
	}

	.payments-actions-btn {
		font-size: 1rem !important;
	}
}

/* Remove readonly styling */
.v-text-field--readonly {
	cursor: text;
}

.v-text-field--readonly:hover {
	background-color: transparent;
}

.cards {
	background-color: rgb(var(--v-theme-surface)) !important;
}

.submit-btn {
	position: relative;
}

.submit-btn:hover,
.submit-btn:focus,
.submit-btn:focus-visible,
.submit-btn:active {
	background-color: rgb(var(--v-theme-primary)) !important;
	color: rgb(var(--v-theme-on-primary)) !important;
	box-shadow: none;
}

.submit-btn:focus-visible {
	outline: 2px solid rgb(var(--v-theme-primary));
	outline-offset: 2px;
}

.submit-btn::before,
.submit-btn:hover::before,
.submit-btn:focus::before,
.submit-btn:focus-visible::before,
.submit-btn:active::before {
	opacity: 0 !important;
}

.submit-highlight {
	box-shadow: 0 0 0 4px rgb(var(--v-theme-primary));
	transition: box-shadow 0.3s ease-in-out;
}

.icon-close-btn {
	background-color: rgba(var(--v-theme-primary), 0.12) !important;
	color: rgb(var(--v-theme-primary)) !important;
	border-radius: 14px !important;
	width: 44px !important;
	height: 44px !important;
}
</style>
