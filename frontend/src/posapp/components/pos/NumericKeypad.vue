<template>
	<v-dialog v-model="internalVisible" persistent max-width="360" attach="body" transition="dialog-bottom-transition">
		<v-card class="numeric-keypad-card">
				<v-card-title class="numeric-keypad-title">
					<v-icon size="small" class="mr-2">mdi-dialpad</v-icon>
					<span>{{ title }}</span>
					<v-spacer></v-spacer>
					<v-btn icon="mdi-close" variant="text" class="numeric-keypad-close" @click="handleCancel"></v-btn>
			</v-card-title>
			<v-divider></v-divider>
			<v-card-text class="numeric-keypad-body">
				<div class="numeric-keypad-display">
					{{ displayValue }}
				</div>
				<div v-if="helperText" class="numeric-keypad-helper">
					{{ helperText }}
				</div>
			</v-card-text>
			<v-divider></v-divider>
			<v-card-actions class="numeric-keypad-grid">
				<v-btn
					v-for="btn in keypadButtons"
					:key="btn.key"
					class="numeric-keypad-button"
					:class="{ 'numeric-keypad-button--full': btn.fullWidth }"
					:color="btn.color"
					:variant="btn.variant || 'tonal'"
					:disabled="btn.disabled"
					@click="handleButton(btn)"
				>
					<v-icon v-if="btn.icon" size="small">{{ btn.icon }}</v-icon>
					<span v-else>{{ btn.label }}</span>
				</v-btn>
			</v-card-actions>
		</v-card>
	</v-dialog>
</template>

<script>
export default {
	name: "NumericKeypad",
	props: {
		modelValue: {
			type: [String, Number],
			default: "0",
		},
		visible: {
			type: Boolean,
			default: false,
		},
		title: {
			type: String,
			default: "Enter Amount",
		},
		helperText: {
			type: String,
			default: "",
		},
		allowDecimal: {
			type: Boolean,
			default: true,
		},
		allowNegative: {
			type: Boolean,
			default: false,
		},
		decimalPlaces: {
			type: Number,
			default: 2,
		},
	},
	emits: ["update:modelValue", "update:visible", "cancel", "confirm"],
	data() {
		return {
			internalVisible: this.visible,
			internalValue: this.normalizeValue(this.modelValue),
		};
	},
	computed: {
		displayValue() {
			if (this.internalValue === "" || this.internalValue === null) {
				return "0";
			}
			return this.internalValue;
		},
		keypadButtons() {
			const baseButtons = [
				{ key: "1", label: "1" },
				{ key: "2", label: "2" },
				{ key: "3", label: "3" },
				{ key: "4", label: "4" },
				{ key: "5", label: "5" },
				{ key: "6", label: "6" },
				{ key: "7", label: "7" },
				{ key: "8", label: "8" },
				{ key: "9", label: "9" },
				{ key: "dot", label: this.allowDecimal ? "." : "00", disabled: !this.allowDecimal && !this.allowZeroPad },
				{ key: "0", label: "0" },
				{ key: "00", label: "00", disabled: false },
			];
			const controls = [
				{ key: "backspace", icon: "mdi-backspace", action: "backspace", variant: "outlined" },
				{ key: "negative", icon: "mdi-minus", action: "toggle-negative", variant: "outlined", disabled: !this.allowNegative },
				{ key: "clear", label: "Clear", action: "clear", color: "warning", variant: "tonal" },
				{ key: "confirm", label: "OK", action: "confirm", color: "primary", variant: "elevated", fullWidth: true },
			];
			return [...baseButtons, ...controls];
		},
		allowZeroPad() {
			return true;
		},
	},
	watch: {
		visible(val) {
			this.internalVisible = val;
			if (val) {
				this.internalValue = this.normalizeValue(this.modelValue);
			}
		},
		internalVisible(val) {
			if (!val) {
				this.$emit("update:visible", false);
			}
		},
		modelValue(val) {
			this.internalValue = this.normalizeValue(val);
		},
	},
	methods: {
		normalizeValue(value) {
			if (value === null || value === undefined) {
				return "0";
			}
			let str = String(value);
			str = str.replace(/[^0-9+\-\.]/g, "");
			if (!this.allowDecimal) {
				str = str.replace(/\./g, "");
			}
			if (!this.allowNegative) {
				str = str.replace(/-/g, "");
			}
			if (str === "" || str === "-") {
				return "0";
			}
			return str;
		},
		handleButton(btn) {
			if (btn.action) {
				switch (btn.action) {
					case "backspace":
						this.backspace();
						break;
					case "toggle-negative":
						this.toggleNegative();
						break;
					case "clear":
						this.clearValue();
						break;
					case "confirm":
						this.confirmValue();
						break;
					default:
						break;
				}
				return;
			}
			this.appendInput(btn.label);
		},
		appendInput(token) {
			let current = this.internalValue || "0";
			if (token === ".") {
				if (!this.allowDecimal || current.includes(".")) {
					return;
				}
				current += ".";
			} else if (token === "00") {
				if (current === "0") {
					current = "0";
				} else {
					current += "00";
				}
			} else {
				if (current === "0") {
					current = token;
				} else if (current === "-0") {
					current = `-${token}`;
				} else {
					current += token;
				}
			}
			this.internalValue = this.sanitisePrecision(current);
			this.$emit("update:modelValue", this.internalValue);
		},
		backspace() {
			let current = this.internalValue || "0";
			if (current.length <= 1 || (current.length === 2 && current.startsWith("-"))) {
				current = "0";
			} else {
				current = current.slice(0, -1);
			}
			if (current === "-") {
				current = "0";
			}
			this.internalValue = current;
			this.$emit("update:modelValue", this.internalValue);
		},
		toggleNegative() {
			if (!this.allowNegative) {
				return;
			}
			let current = this.internalValue || "0";
			if (current.startsWith("-")) {
				current = current.replace(/^-/, "");
			} else {
				current = `-${current}`;
			}
			this.internalValue = current;
			this.$emit("update:modelValue", this.internalValue);
		},
		clearValue() {
			this.internalValue = "0";
			this.$emit("update:modelValue", this.internalValue);
		},
		confirmValue() {
			const sanitized = this.sanitisePrecision(this.internalValue);
			this.internalValue = sanitized;
			this.$emit("update:modelValue", sanitized);
			this.$emit("confirm", sanitized);
			this.close();
		},
		handleCancel() {
			this.close();
			this.$emit("cancel");
		},
		close() {
			this.internalVisible = false;
			this.$emit("update:visible", false);
		},
		sanitisePrecision(value) {
			if (!this.allowDecimal) {
				return value;
			}
			const [intPart, decimalPart] = value.split(".");
			if (decimalPart && decimalPart.length > this.decimalPlaces) {
				return `${intPart}.${decimalPart.slice(0, this.decimalPlaces)}`;
			}
			return value;
		},
	},
};
</script>

<style scoped>
.numeric-keypad-card {
	border-radius: 20px;
	overflow: hidden;
}

.numeric-keypad-title {
	display: flex;
	align-items: center;
	gap: 10px;
	font-weight: 600;
}

.numeric-keypad-body {
	display: flex;
	flex-direction: column;
	align-items: center;
	gap: 8px;
	padding: 16px;
}

.numeric-keypad-display {
	width: 100%;
	padding: 16px;
	border-radius: 12px;
	text-align: right;
	font-size: 1.8rem;
	font-weight: 600;
	background-color: rgba(var(--v-theme-surface-variant), 0.35);
	color: rgb(var(--v-theme-on-surface));
	letter-spacing: 0.04em;
}

.numeric-keypad-helper {
	font-size: 0.85rem;
	color: rgba(var(--v-theme-on-surface), 0.6);
}

.numeric-keypad-grid {
	display: grid;
	grid-template-columns: repeat(3, minmax(0, 1fr));
	gap: 12px;
	padding: 20px;
}

.numeric-keypad-button {
	font-size: 1.1rem;
	font-weight: 600;
	height: 54px;
	border-radius: 12px !important;
}

.numeric-keypad-button :deep(.v-btn__content) {
	width: 100%;
	justify-content: center;
}

.numeric-keypad-button--full {
	grid-column: span 3;
}

.numeric-keypad-close {
	background-color: rgba(var(--v-theme-primary), 0.12) !important;
	color: rgb(var(--v-theme-primary)) !important;
	border-radius: 12px !important;
	width: 44px !important;
	height: 44px !important;
}
</style>
