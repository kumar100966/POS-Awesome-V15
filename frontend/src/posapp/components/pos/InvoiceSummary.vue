<template>
	<v-card class="cards mb-0 mt-3 py-2 px-3 rounded-lg resizable pos-themed-card" style="resize: vertical; overflow: auto">
		<div class="summary-section summary-totals">
			<v-row dense>

				<!-- Additional Discount (Amount or Percentage) -->
				<v-col cols="12" v-if="!pos_profile.posa_use_percentage_discount">
					<v-text-field :model-value="additional_discount" @update:model-value="handleAdditionalDiscountUpdate" @focus="handleAdditionalDiscountFocus" :label="frappe._('Additional Discount')" prepend-inner-icon="mdi-cash-minus" variant="solo" density="compact" color="warning" :prefix="currencySymbol(pos_profile.currency)" :disabled="!pos_profile.posa_allow_user_to_edit_additional_discount ||
						!!discount_percentage_offer_name
						" class="summary-field summary-field--metric">
						<template #append-inner>
							<v-btn icon size="large" class="touch-keypad-btn" :disabled="!pos_profile.posa_allow_user_to_edit_additional_discount ||
								!!discount_percentage_offer_name
								" @click.stop="openAdditionalDiscountKeypad">
								<v-icon size="28">mdi-dialpad</v-icon>
							</v-btn>
						</template>
					</v-text-field>
				</v-col>

				<v-col cols="12" v-else>
					<v-text-field :model-value="additional_discount_percentage" @update:model-value="handleAdditionalDiscountPercentageUpdate" @change="$emit('update_discount_umount')" @focus="handleAdditionalDiscountPercentageFocus" :rules="[isNumber]" :label="frappe._('Additional Discount %')" suffix="%" prepend-inner-icon="mdi-percent" variant="solo" density="compact" color="warning" :disabled="!pos_profile.posa_allow_user_to_edit_additional_discount ||
						!!discount_percentage_offer_name
						" class="summary-field summary-field--metric">
						<template #append-inner>
							<v-btn icon size="large" class="touch-keypad-btn" :disabled="!pos_profile.posa_allow_user_to_edit_additional_discount ||
								!!discount_percentage_offer_name
								" @click.stop="openAdditionalDiscountPercentageKeypad">
								<v-icon size="28">mdi-dialpad</v-icon>
							</v-btn>
						</template>
					</v-text-field>
				</v-col>

				<!-- Total -->
				<v-col cols="12">
					<v-text-field :model-value="formatCurrency(subtotal)" :prefix="currencySymbol(displayCurrency)" :label="frappe._('Total')" prepend-inner-icon="mdi-cash" variant="solo" density="compact" readonly color="success" class="summary-field summary-field--metric" />
				</v-col>

			</v-row>
		</div>

		<v-divider class="summary-divider" />

		<div class="summary-section summary-actions">
			<v-row dense>
				<v-col cols="12">
					<v-btn block color="accent" theme="dark" prepend-icon="mdi-content-save" @click="handleSaveAndClear" class="summary-btn" :loading="saveLoading">
						{{ __("Save & Clear") }}
					</v-btn>
				</v-col>
				<v-col cols="12">
					<v-btn block color="warning" theme="dark" prepend-icon="mdi-file-document" @click="handleLoadDrafts" class="white-text-btn summary-btn" :loading="loadDraftsLoading">
						{{ __("Load Drafts") }}
					</v-btn>
				</v-col>
				<v-col cols="12" v-if="pos_profile.custom_allow_select_sales_order == 1">
					<v-btn block color="info" theme="dark" prepend-icon="mdi-book-search" @click="handleSelectOrder" class="summary-btn" :loading="selectOrderLoading">
						{{ __("Select S.O") }}
					</v-btn>
				</v-col>
				<v-col cols="12">
					<v-btn block color="error" theme="dark" prepend-icon="mdi-close-circle" @click="handleCancelSale" class="summary-btn" :loading="cancelLoading">
						{{ __("Cancel Sale") }}
					</v-btn>
				</v-col>
				<v-col cols="12" v-if="pos_profile.posa_allow_return == 1">
					<v-btn block color="secondary" theme="dark" prepend-icon="mdi-backup-restore" @click="handleOpenReturns" class="summary-btn" :loading="returnsLoading">
						{{ __("Sales Return") }}
					</v-btn>
				</v-col>
				<v-col cols="12" v-if="pos_profile.posa_allow_print_draft_invoices">
					<v-btn block color="primary" theme="dark" prepend-icon="mdi-printer" @click="handlePrintDraft" class="summary-btn" :loading="printLoading">
						{{ __("Print Draft") }}
					</v-btn>
				</v-col>
				<v-col cols="12">
					<v-btn block color="success" theme="dark" size="large" prepend-icon="mdi-credit-card" @click="handleShowPayment" class="summary-btn pay-btn" :loading="paymentLoading">
						{{ __("PAY") }}
					</v-btn>
				</v-col>
			</v-row>
		</div>
	</v-card>
	<NumericKeypad :visible="numericKeypad.visible" :model-value="numericKeypad.value" :title="numericKeypad.title" :helper-text="numericKeypad.helperText" :allow-decimal="numericKeypad.allowDecimal" :allow-negative="numericKeypad.allowNegative" :decimal-places="numericKeypad.decimalPlaces" @update:modelValue="(val) => (numericKeypad.value = val)" @update:visible="(val) => (numericKeypad.visible = val)" @confirm="handleNumericKeypadConfirm" @cancel="closeNumericKeypad" />
</template>

<script>
import { formatUtils } from "../../format";
import NumericKeypad from "./NumericKeypad.vue";

export default {
	components: {
		NumericKeypad,
	},
	props: {
		pos_profile: Object,
		total_qty: [Number, String],
		additional_discount: Number,
		additional_discount_percentage: Number,
		total_items_discount_amount: Number,
		subtotal: [Number, String],
		taxTotal: {
			type: [Number, String],
			default: 0,
		},
		grandTotal: {
			type: [Number, String],
			default: 0,
		},
		displayCurrency: String,
		formatFloat: Function,
		formatCurrency: Function,
		currencySymbol: Function,
		discount_percentage_offer_name: [String, Number],
		isNumber: Function,
	},
	data() {
		return {
			// Loading states for better UX
			saveLoading: false,
			loadDraftsLoading: false,
			selectOrderLoading: false,
			cancelLoading: false,
			returnsLoading: false,
			printLoading: false,
			paymentLoading: false,
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
	emits: [
		"update:additional_discount",
		"update:additional_discount_percentage",
		"update_discount_umount",
		"save-and-clear",
		"load-drafts",
		"select-order",
		"cancel-sale",
		"open-returns",
		"print-draft",
		"show-payment",
	],
	computed: {
		hide_qty_decimals() {
			try {
				const saved = localStorage.getItem("posawesome_item_selector_settings");
				if (saved) {
					const opts = JSON.parse(saved);
					return !!opts.hide_qty_decimals;
				}
			} catch (e) {
				console.error("Failed to load item selector settings:", e);
			}
			return false;
		},
	},
	methods: {
		openNumericKeypad({
			initialValue = 0,
			title = __("Enter Value"),
			helperText = "",
			allowDecimal = true,
			allowNegative = false,
			decimalPlaces = null,
			onConfirm = null,
		}) {
			const defaultPrecision = Number(this.pos_profile?.currency_precision ?? 2);
			const precision = Number.isFinite(Number(decimalPlaces))
				? Number(decimalPlaces)
				: defaultPrecision;
			this.numericKeypad.title = title;
			this.numericKeypad.helperText = helperText;
			this.numericKeypad.allowDecimal = allowDecimal;
			this.numericKeypad.allowNegative = allowNegative;
			this.numericKeypad.decimalPlaces = precision;
			this.numericKeypad.apply = typeof onConfirm === "function" ? onConfirm : null;
			this.numericKeypad.value = this.prepareKeypadValue(initialValue, precision);
			this.numericKeypad.visible = true;
		},
		closeNumericKeypad() {
			this.numericKeypad.visible = false;
			this.numericKeypad.apply = null;
		},
		handleNumericKeypadConfirm(rawValue) {
			const precision = this.numericKeypad.decimalPlaces ?? Number(this.pos_profile?.currency_precision ?? 2);
			const numericValue = this.parseKeypadNumber(rawValue, precision);
			if (typeof this.numericKeypad.apply === "function") {
				this.numericKeypad.apply(numericValue);
			}
			this.closeNumericKeypad();
		},
		prepareKeypadValue(value, precision = 2) {
			const numeric = this.parseKeypadNumber(value, precision);
			const fixed = precision > 0 ? Number(numeric.toFixed(precision)) : numeric;
			return String(fixed).replace(/^-0$/, "0");
		},
		parseKeypadNumber(value, precision = 2) {
			const parsed = parseFloat(formatUtils.fromArabicNumerals(String(value ?? 0)));
			if (Number.isNaN(parsed)) {
				return 0;
			}
			const factor = Math.pow(10, precision);
			return Math.round(parsed * factor) / factor;
		},
		openAdditionalDiscountKeypad() {
			const precision = Number(this.pos_profile?.currency_precision ?? 2);
			this.openNumericKeypad({
				initialValue: this.additional_discount || 0,
				title: __("Set Additional Discount"),
				decimalPlaces: precision,
				onConfirm: (value) => {
					const sanitized = Math.max(0, value);
					this.handleAdditionalDiscountUpdate(sanitized);
				},
			});
		},
		openAdditionalDiscountPercentageKeypad() {
			this.openNumericKeypad({
				initialValue: this.additional_discount_percentage || 0,
				title: __("Set Additional Discount %"),
				decimalPlaces: 2,
				onConfirm: (value) => {
					const sanitized = Math.max(0, Math.min(100, value));
					this.handleAdditionalDiscountPercentageUpdate(sanitized);
					this.$emit("update_discount_umount");
				},
			});
		},
		handleAdditionalDiscountFocus() {
			if (this.prefersTouchKeypad) {
				this.openAdditionalDiscountKeypad();
			}
		},
		handleAdditionalDiscountPercentageFocus() {
			if (this.prefersTouchKeypad) {
				this.openAdditionalDiscountPercentageKeypad();
			}
		},
		// Debounced handlers for better performance
		handleAdditionalDiscountUpdate(value) {
			this.$emit("update:additional_discount", value);
		},

		handleAdditionalDiscountPercentageUpdate(value) {
			this.$emit("update:additional_discount_percentage", value);
		},

		async handleSaveAndClear() {
			this.saveLoading = true;
			try {
				await this.$emit("save-and-clear");
			} finally {
				this.saveLoading = false;
			}
		},

		async handleLoadDrafts() {
			this.loadDraftsLoading = true;
			try {
				await this.$emit("load-drafts");
			} finally {
				this.loadDraftsLoading = false;
			}
		},

		async handleSelectOrder() {
			this.selectOrderLoading = true;
			try {
				await this.$emit("select-order");
			} finally {
				this.selectOrderLoading = false;
			}
		},

		async handleCancelSale() {
			this.cancelLoading = true;
			try {
				await this.$emit("cancel-sale");
			} finally {
				this.cancelLoading = false;
			}
		},

		async handleOpenReturns() {
			this.returnsLoading = true;
			try {
				await this.$emit("open-returns");
			} finally {
				this.returnsLoading = false;
			}
		},

		async handlePrintDraft() {
			this.printLoading = true;
			try {
				await this.$emit("print-draft");
			} finally {
				this.printLoading = false;
			}
		},

		async handleShowPayment() {
			this.paymentLoading = true;
			try {
				await this.$emit("show-payment");
			} finally {
				this.paymentLoading = false;
			}
		},
	},
	mounted() {
		if (typeof window !== "undefined") {
			try {
				const coarse = window.matchMedia?.("(pointer: coarse)")?.matches;
				this.prefersTouchKeypad = Boolean(coarse || "ontouchstart" in window);
			} catch (err) {
				this.prefersTouchKeypad = false;
			}
		}
	},
	beforeUnmount() {
		this.closeNumericKeypad();
	},
};
</script>

<style scoped>
.cards {
	background-color: var(--pos-card-bg) !important;
	transition: all 0.3s ease;
}

.white-text-btn {
	color: var(--pos-text-primary) !important;
}

.white-text-btn :deep(.v-btn__content) {
	color: var(--pos-text-primary) !important;
}

/* Enhanced button styling with better performance */
.summary-btn {
	transition: all 0.2s ease !important;
	position: relative;
	overflow: hidden;
	color: #fff !important;
	min-height: 56px !important;
	font-size: 1.05rem !important;
	padding: 14px 18px !important;
	border-radius: 14px !important;
}

.summary-btn :deep(.v-btn__content) {
	white-space: normal !important;
	transition: all 0.2s ease;
	color: inherit !important;
	font-weight: 600;
}

.summary-btn:hover {
	transform: translateY(-1px);
	box-shadow: 0 4px 8px rgba(0, 0, 0, 0.15) !important;
}

.summary-btn:hover,
.summary-btn:focus-visible {
	color: #fff !important;
}

.summary-btn:hover :deep(.v-btn__content),
.summary-btn:focus-visible :deep(.v-btn__content) {
	color: inherit !important;
}

.white-text-btn.summary-btn,
.white-text-btn.summary-btn :deep(.v-btn__content) {
	color: var(--pos-text-primary) !important;
}

.summary-btn:active {
	transform: translateY(0);
}

/* Special styling for the PAY button */
.pay-btn {
	font-weight: 600 !important;
	font-size: 1.1rem !important;
	background: linear-gradient(135deg, #4caf50, #45a049) !important;
	box-shadow: 0 4px 12px rgba(76, 175, 80, 0.3) !important;
}

.pay-btn:hover {
	background: linear-gradient(135deg, #45a049, #3d8b40) !important;
	box-shadow: 0 6px 16px rgba(76, 175, 80, 0.4) !important;
	transform: translateY(-2px);
}

/* Enhanced field styling */
.summary-field {
	transition: all 0.2s ease;
}

.summary-field:hover {
	transform: translateY(-1px);
	box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.touch-keypad-btn {
	min-width: 44px !important;
	height: 44px !important;
	border-radius: 12px !important;
}

.touch-keypad-btn :deep(.v-icon) {
	font-size: 28px !important;
}

.summary-field--metric :deep(.v-field__input) {
	font-size: 1.25rem !important;
	font-weight: 600 !important;
}

.summary-field--metric :deep(.v-field__label) {
	font-size: 1rem !important;
	font-weight: 500 !important;
}

.summary-metric-col {
	margin-bottom: 6px;
}

.summary-metric-card {
	display: flex;
	flex-direction: column;
	align-items: flex-start;
	gap: 6px;
	padding: 16px 18px;
	border-radius: 14px;
	background: rgba(var(--v-theme-surface), 0.95);
	border: 1px solid rgba(var(--v-theme-outline), 0.16);
	box-shadow: 0 10px 24px rgba(15, 23, 42, 0.08);
	width: 100%;
}

.summary-metric-card__icon {
	color: rgba(var(--v-theme-primary));
}

.summary-metric-card__label {
	font-size: 0.95rem;
	font-weight: 500;
	color: rgba(var(--v-theme-on-surface), 0.68);
}

.summary-metric-card__value {
	font-size: 2rem;
	font-weight: 700;
	color: rgba(var(--v-theme-primary));
	letter-spacing: -0.03em;
}

.summary-section {
	display: flex;
	flex-direction: column;
	gap: var(--dynamic-sm);
}

.summary-totals :deep(.v-row) {
	display: flex;
	flex-direction: column;
	row-gap: 10px;
}

.summary-section :deep(.v-row) {
	margin: 0;
}

.summary-section :deep(.v-col) {
	padding: 4px 6px;
}

.summary-divider {
	margin: var(--dynamic-sm) 0;
}

.summary-actions :deep(.v-col) {
	display: flex;
}

.summary-actions :deep(.v-btn) {
	flex: 1 1 auto;
}

/* Responsive optimizations */
@media (max-width: 768px) {
	.summary-btn {
		font-size: 0.875rem !important;
		padding: 8px 12px !important;
	}

	.pay-btn {
		font-size: 1rem !important;
	}

	.summary-field {
		font-size: 0.875rem;
	}
}

@media (max-width: 480px) {
	.summary-btn {
		font-size: 0.8rem !important;
		padding: 6px 8px !important;
	}

	.pay-btn {
		font-size: 0.95rem !important;
	}
}

/* Loading state animations */
.summary-btn:deep(.v-btn__loader) {
	opacity: 0.8;
}

/* Dark theme enhancements */
:deep([data-theme="dark"]) .summary-btn,
:deep(.v-theme--dark) .summary-btn {
	box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3) !important;
}

:deep([data-theme="dark"]) .summary-btn:hover,
:deep(.v-theme--dark) .summary-btn:hover {
	box-shadow: 0 4px 12px rgba(0, 0, 0, 0.4) !important;
}
</style>
