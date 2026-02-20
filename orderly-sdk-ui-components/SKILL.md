# Orderly SDK UI Components

A comprehensive guide to all UI components available in the Orderly Network SDK for building decentralized exchange applications.

## Overview

The Orderly SDK provides a complete UI component library built with React, Radix UI primitives, and Tailwind CSS. Components are organized into two main categories:

1. **Base UI Components** (`@orderly.network/ui`) - Foundational UI primitives like buttons, inputs, dialogs, tables
2. **Domain-Specific Components** (`@orderly.network/ui-*`) - Trading-specific widgets like order entry, positions, charts

## Installation

### Base UI Package

```bash
npm install @orderly.network/ui
# or
yarn add @orderly.network/ui
# or
pnpm add @orderly.network/ui
```

### Domain-Specific Packages

```bash
# Order Entry
npm install @orderly.network/ui-order-entry

# Positions Management
npm install @orderly.network/ui-positions

# Orders Management
npm install @orderly.network/ui-orders

# Leverage Controls
npm install @orderly.network/ui-leverage

# Take-Profit/Stop-Loss
npm install @orderly.network/ui-tpsl

# Deposit/Withdraw/Transfer
npm install @orderly.network/ui-transfer

# Wallet Connection
npm install @orderly.network/ui-connector

# Chain Selection
npm install @orderly.network/ui-chain-selector

# TradingView Integration
npm install @orderly.network/ui-tradingview

# Share PnL
npm install @orderly.network/ui-share

# App Scaffolding
npm install @orderly.network/ui-scaffold
```

## Setup

### Import Styles

```tsx
import "@orderly.network/ui/dist/styles.css";
```

### Theme Provider

```tsx
import { OrderlyThemeProvider } from "@orderly.network/ui";

function App() {
  return (
    <OrderlyThemeProvider>
      <YourApp />
    </OrderlyThemeProvider>
  );
}
```

---

# Base UI Components (`@orderly.network/ui`)

## Layout Components

### Box

A polymorphic container component with flexbox utilities.

```tsx
import { Box } from "@orderly.network/ui";

<Box p={4} m={2} r="lg" className="custom-class">
  Content
</Box>
```

**Props:**
| Prop | Type | Description |
|------|------|-------------|
| `p` | `number \| string` | Padding |
| `px`, `py` | `number \| string` | Horizontal/vertical padding |
| `m` | `number \| string` | Margin |
| `r` | `"sm" \| "md" \| "lg" \| "xl"` | Border radius |
| `intensity` | `number` | Background intensity |

---

### Flex

Flexbox layout container.

```tsx
import { Flex } from "@orderly.network/ui";

<Flex direction="row" gap={2} justify="between" align="center">
  <div>Item 1</div>
  <div>Item 2</div>
</Flex>
```

**Props:**
| Prop | Type | Description |
|------|------|-------------|
| `direction` | `"row" \| "column"` | Flex direction |
| `gap` | `number` | Gap between items |
| `justify` | `"start" \| "center" \| "end" \| "between" \| "around"` | Justify content |
| `align` | `"start" \| "center" \| "end" \| "stretch"` | Align items |
| `wrap` | `boolean` | Flex wrap |

---

### Grid

CSS Grid layout container.

```tsx
import { Grid } from "@orderly.network/ui";

<Grid cols={3} gap={4}>
  <div>Cell 1</div>
  <div>Cell 2</div>
  <div>Cell 3</div>
</Grid>
```

---

## Form Components

### Button

Customizable button with multiple variants and colors.

```tsx
import { Button } from "@orderly.network/ui";

<Button 
  variant="contained" 
  color="primary" 
  size="md"
  fullWidth
  loading={isLoading}
  disabled={isDisabled}
  onClick={handleClick}
>
  Submit Order
</Button>
```

**Props:**
| Prop | Type | Description |
|------|------|-------------|
| `variant` | `"text" \| "outlined" \| "contained" \| "gradient"` | Button style variant |
| `color` | `"primary" \| "secondary" \| "success" \| "danger" \| "warning" \| "buy" \| "sell" \| "gray"` | Button color |
| `size` | `"xs" \| "sm" \| "md" \| "lg" \| "xl"` | Button size |
| `fullWidth` | `boolean` | Full width button |
| `loading` | `boolean` | Show loading spinner |
| `disabled` | `boolean` | Disable button |
| `asChild` | `boolean` | Render as child element |

**Size Reference:**
- `xs`: 24px height
- `sm`: 28px height
- `md`: 32px height
- `lg`: 40px height
- `xl`: 54px height

---

### Input

Text input with formatting and validation support.

```tsx
import { Input, TextField, inputFormatter } from "@orderly.network/ui";

// Basic Input
<Input
  placeholder="Enter amount"
  size="md"
  color="default"
  disabled={false}
/>

// With Formatter (numeric input)
<Input
  placeholder="0.00"
  formatters={[inputFormatter.numberFormatter]}
  onValueChange={(value) => setValue(value)}
/>

// TextField with Label
<TextField
  label="Price"
  placeholder="Enter price"
  suffix="USDC"
  error="Invalid price"
/>
```

**Props:**
| Prop | Type | Description |
|------|------|-------------|
| `size` | `"xs" \| "sm" \| "md" \| "lg" \| "xl"` | Input size |
| `color` | `"default" \| "success" \| "danger" \| "warning"` | Input color state |
| `disabled` | `boolean` | Disable input |
| `prefix` | `ReactNode` | Prefix element |
| `suffix` | `ReactNode` | Suffix element |
| `formatters` | `InputFormatter[]` | Value formatters |
| `onValueChange` | `(value: string) => void` | Value change handler |

**Built-in Formatters:**
```tsx
import { inputFormatter } from "@orderly.network/ui";

inputFormatter.numberFormatter     // Numeric values only
inputFormatter.currencyFormatter   // Currency formatting
inputFormatter.dpFormatter(2)      // Decimal places limiter
```

---

### Select

Dropdown select component.

```tsx
import { Select, SelectItem } from "@orderly.network/ui";

<Select 
  value={selected}
  onValueChange={setSelected}
  placeholder="Select option"
  size="md"
>
  <SelectItem value="option1">Option 1</SelectItem>
  <SelectItem value="option2">Option 2</SelectItem>
  <SelectItem value="option3">Option 3</SelectItem>
</Select>
```

**Props:**
| Prop | Type | Description |
|------|------|-------------|
| `value` | `string` | Selected value |
| `defaultValue` | `string` | Default value |
| `onValueChange` | `(value: string) => void` | Change handler |
| `placeholder` | `string` | Placeholder text |
| `size` | `"xs" \| "sm" \| "md" \| "lg"` | Select size |
| `error` | `boolean` | Error state |
| `showCaret` | `boolean` | Show dropdown arrow |
| `maxHeight` | `number` | Max dropdown height |

---

### Checkbox

Checkbox input component.

```tsx
import { Checkbox } from "@orderly.network/ui";

<Checkbox
  checked={isChecked}
  onCheckedChange={setIsChecked}
  id="terms"
>
  Accept terms and conditions
</Checkbox>
```

---

### Switch

Toggle switch component.

```tsx
import { Switch } from "@orderly.network/ui";

<Switch
  checked={isEnabled}
  onCheckedChange={setIsEnabled}
  size="default"
/>
```

---

### Slider

Range slider with marks and custom styling.

```tsx
import { Slider } from "@orderly.network/ui";

<Slider
  value={[50]}
  onValueChange={(values) => setValue(values[0])}
  min={0}
  max={100}
  step={1}
  color="primary"
  marks={[
    { value: 0, label: "0%" },
    { value: 25, label: "25%" },
    { value: 50, label: "50%" },
    { value: 75, label: "75%" },
    { value: 100, label: "100%" },
  ]}
/>
```

**Props:**
| Prop | Type | Description |
|------|------|-------------|
| `value` | `number[]` | Current value(s) |
| `onValueChange` | `(values: number[]) => void` | Change handler |
| `min` | `number` | Minimum value |
| `max` | `number` | Maximum value |
| `step` | `number` | Step increment |
| `color` | `"primary" \| "primaryLight" \| "buy" \| "sell"` | Slider color |
| `marks` | `SliderMarks[]` | Mark points |
| `disabled` | `boolean` | Disable slider |

---

## Typography

### Text

Flexible text component with styling variants.

```tsx
import { Text } from "@orderly.network/ui";

<Text 
  size="sm" 
  weight="semibold" 
  color="primary"
  intensity={80}
  as="span"
>
  Account Balance
</Text>
```

**Props:**
| Prop | Type | Description |
|------|------|-------------|
| `size` | `"3xs" \| "2xs" \| "xs" \| "sm" \| "base" \| "lg" \| "xl" \| "2xl" \| "3xl"` | Font size |
| `weight` | `"regular" \| "semibold" \| "bold"` | Font weight |
| `color` | `"inherit" \| "neutral" \| "primary" \| "secondary" \| "warning" \| "danger" \| "success" \| "buy" \| "sell"` | Text color |
| `intensity` | `12 \| 20 \| 36 \| 54 \| 80 \| 98` | Opacity intensity |
| `as` | `"span" \| "div" \| "p" \| "label"` | HTML element |
| `copyable` | `boolean` | Enable copy on click |

---

### Numeral

Formatted numeric display.

```tsx
import { Numeral } from "@orderly.network/ui";

<Numeral value={12345.67} dp={2} prefix="$" />
// Renders: $12,345.67

<Numeral 
  value={0.0534} 
  dp={4} 
  suffix="%" 
  coloring  // Green for positive, red for negative
/>
```

---

## Overlay Components

### Dialog

Modal dialog component.

```tsx
import { 
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
  DialogBody,
  DialogFooter,
  DialogClose
} from "@orderly.network/ui";

<Dialog open={isOpen} onOpenChange={setIsOpen}>
  <DialogContent>
    <DialogHeader>
      <DialogTitle>Confirm Order</DialogTitle>
    </DialogHeader>
    <DialogBody>
      Are you sure you want to place this order?
    </DialogBody>
    <DialogFooter>
      <DialogClose asChild>
        <Button variant="outlined">Cancel</Button>
      </DialogClose>
      <Button color="primary" onClick={handleConfirm}>
        Confirm
      </Button>
    </DialogFooter>
  </DialogContent>
</Dialog>
```

**DialogContent Props:**
| Prop | Type | Description |
|------|------|-------------|
| `size` | `"sm" \| "md" \| "lg"` | Dialog size |
| `closable` | `boolean` | Show close button |
| `onCloseAutoFocus` | `(e: Event) => void` | Close focus handler |

---

### Sheet

Bottom/side sheet component.

```tsx
import { 
  Sheet,
  SheetContent,
  SheetHeader,
  SheetTitle,
  SheetTrigger
} from "@orderly.network/ui";

<Sheet>
  <SheetTrigger asChild>
    <Button>Open Menu</Button>
  </SheetTrigger>
  <SheetContent side="bottom">
    <SheetHeader>
      <SheetTitle>Options</SheetTitle>
    </SheetHeader>
    {/* Content */}
  </SheetContent>
</Sheet>
```

**SheetContent Props:**
| Prop | Type | Description |
|------|------|-------------|
| `side` | `"top" \| "right" \| "bottom" \| "left"` | Sheet position |

---

### Modal

Programmatic modal system.

```tsx
import { modal, ModalProvider } from "@orderly.network/ui";

// Wrap app with ModalProvider
<ModalProvider>
  <App />
</ModalProvider>

// Show confirmation dialog
const result = await modal.confirm({
  title: "Close Position",
  content: "Are you sure you want to close this position?",
  onOk: () => { /* handle confirm */ },
  onCancel: () => { /* handle cancel */ },
});

// Show alert
modal.alert({
  title: "Error",
  content: "Insufficient balance",
});

// Show custom dialog
modal.dialog({
  title: "Custom",
  content: <CustomComponent />,
});

// Show sheet
modal.sheet({
  title: "Options",
  content: <OptionsMenu />,
});
```

**Modal Methods:**
| Method | Description |
|--------|-------------|
| `modal.confirm()` | Confirmation dialog with OK/Cancel |
| `modal.alert()` | Alert dialog |
| `modal.dialog()` | Custom dialog |
| `modal.sheet()` | Bottom sheet |
| `modal.create()` | Create custom modal |
| `modal.register()` | Register modal component |

---

### Popover

Floating popover component.

```tsx
import { Popover, PopoverRoot, PopoverTrigger, PopoverContent } from "@orderly.network/ui";

// Simple usage
<Popover content={<div>Popover content</div>} arrow>
  <Button>Hover me</Button>
</Popover>

// Advanced usage
<PopoverRoot>
  <PopoverTrigger asChild>
    <Button>Click me</Button>
  </PopoverTrigger>
  <PopoverContent align="center" sideOffset={4}>
    <div>Popover content</div>
  </PopoverContent>
</PopoverRoot>
```

---

### Tooltip

Tooltip component.

```tsx
import { Tooltip, TooltipContent, TooltipTrigger } from "@orderly.network/ui";

<Tooltip content="This is a tooltip">
  <Button>Hover for info</Button>
</Tooltip>
```

---

### Dropdown

Dropdown menu component.

```tsx
import { 
  DropdownMenu,
  DropdownMenuTrigger,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuSeparator
} from "@orderly.network/ui";

<DropdownMenu>
  <DropdownMenuTrigger asChild>
    <Button>Options</Button>
  </DropdownMenuTrigger>
  <DropdownMenuContent size="md">
    <DropdownMenuItem onClick={() => handleEdit()}>
      Edit
    </DropdownMenuItem>
    <DropdownMenuItem onClick={() => handleDuplicate()}>
      Duplicate
    </DropdownMenuItem>
    <DropdownMenuSeparator />
    <DropdownMenuItem onClick={() => handleDelete()}>
      Delete
    </DropdownMenuItem>
  </DropdownMenuContent>
</DropdownMenu>
```

---

## Data Display

### DataTable

Powerful data table with sorting, filtering, and pagination.

```tsx
import { DataTable, type DataTableProps } from "@orderly.network/ui";

const columns = [
  {
    id: "symbol",
    header: "Symbol",
    accessorKey: "symbol",
  },
  {
    id: "price",
    header: "Price",
    accessorKey: "price",
    cell: ({ getValue }) => <Numeral value={getValue()} dp={2} />,
  },
  {
    id: "change",
    header: "24h Change",
    accessorKey: "change24h",
    cell: ({ getValue }) => (
      <Text color={getValue() >= 0 ? "success" : "danger"}>
        {getValue()}%
      </Text>
    ),
  },
];

<DataTable
  columns={columns}
  data={marketData}
  loading={isLoading}
  pagination={{
    page: currentPage,
    pageSize: 10,
    total: totalCount,
    onPageChange: setCurrentPage,
  }}
  onSort={handleSort}
  emptyView={<EmptyDataState message="No data available" />}
/>
```

**Props:**
| Prop | Type | Description |
|------|------|-------------|
| `columns` | `ColumnDef[]` | Column definitions |
| `data` | `T[]` | Table data |
| `loading` | `boolean` | Loading state |
| `pagination` | `PaginationProps` | Pagination config |
| `onSort` | `(sortState) => void` | Sort handler |
| `emptyView` | `ReactNode` | Empty state component |
| `rowClassName` | `(row) => string` | Row class generator |
| `onRowClick` | `(row) => void` | Row click handler |

---

### Tabs

Tabbed interface component.

```tsx
import { Tabs, TabsList, TabsTrigger, TabsContent } from "@orderly.network/ui";

<Tabs defaultValue="positions" variant="contained">
  <TabsList>
    <TabsTrigger value="positions">Positions</TabsTrigger>
    <TabsTrigger value="orders">Orders</TabsTrigger>
    <TabsTrigger value="history">History</TabsTrigger>
  </TabsList>
  <TabsContent value="positions">
    <PositionsList />
  </TabsContent>
  <TabsContent value="orders">
    <OrdersList />
  </TabsContent>
  <TabsContent value="history">
    <TradeHistory />
  </TabsContent>
</Tabs>
```

**Props:**
| Prop | Type | Description |
|------|------|-------------|
| `defaultValue` | `string` | Default active tab |
| `value` | `string` | Controlled active tab |
| `onValueChange` | `(value: string) => void` | Tab change handler |
| `variant` | `"text" \| "contained"` | Tab style variant |

---

### Card

Card container component.

```tsx
import { Card, CardHeader, CardContent, CardFooter } from "@orderly.network/ui";

<Card>
  <CardHeader>
    <Text size="lg" weight="semibold">Account Summary</Text>
  </CardHeader>
  <CardContent>
    <Flex direction="column" gap={2}>
      <Text>Total Equity: $10,000</Text>
      <Text>Free Collateral: $5,000</Text>
    </Flex>
  </CardContent>
  <CardFooter>
    <Button fullWidth>Deposit</Button>
  </CardFooter>
</Card>
```

---

### Badge

Status badge component.

```tsx
import { Badge } from "@orderly.network/ui";

<Badge color="success">Active</Badge>
<Badge color="warning">Pending</Badge>
<Badge color="danger">Failed</Badge>
```

---

### Divider

Horizontal/vertical divider.

```tsx
import { Divider } from "@orderly.network/ui";

<Divider />
<Divider orientation="vertical" />
<Divider className="oui-my-4" />
```

---

## Feedback Components

### Toast

Toast notification system.

```tsx
import { toast, Toaster } from "@orderly.network/ui";

// Add Toaster to app root
<Toaster />

// Show toasts
toast.success("Order placed successfully!");
toast.error("Failed to place order");
toast.loading("Processing...");

// Custom toast
toast.custom((t) => (
  <div>
    Custom notification
    <button onClick={() => toast.dismiss(t.id)}>Close</button>
  </div>
));
```

**Toast Methods:**
| Method | Description |
|--------|-------------|
| `toast.success(message)` | Success notification |
| `toast.error(message)` | Error notification |
| `toast.loading(message)` | Loading notification |
| `toast.custom(render)` | Custom notification |
| `toast.dismiss(id)` | Dismiss specific toast |

---

### Spinner

Loading spinner component.

```tsx
import { Spinner } from "@orderly.network/ui";

<Spinner size="sm" />
<Spinner size="md" />
<Spinner size="lg" />
```

---

### EmptyDataState

Empty state placeholder.

```tsx
import { EmptyDataState } from "@orderly.network/ui";

<EmptyDataState 
  message="No positions found"
  icon={<EmptyIcon />}
/>
```

---

## Utility Components

### ScrollArea

Custom scrollable container.

```tsx
import { ScrollArea } from "@orderly.network/ui";

<ScrollArea className="h-[300px]">
  {/* Scrollable content */}
</ScrollArea>
```

---

### Collapsible

Expandable/collapsible section.

```tsx
import { Collapsible, CollapsibleTrigger, CollapsibleContent } from "@orderly.network/ui";

<Collapsible>
  <CollapsibleTrigger>
    <Text>Show Details</Text>
  </CollapsibleTrigger>
  <CollapsibleContent>
    <div>Hidden details content</div>
  </CollapsibleContent>
</Collapsible>
```

---

### Avatar

User/entity avatar component.

```tsx
import { Avatar, EVMAvatar } from "@orderly.network/ui";

<Avatar src="/avatar.png" size="md" />

// Ethereum blockie avatar
<EVMAvatar address="0x1234..." size={32} />
```

---

### Logo

Orderly logo component.

```tsx
import { Logo } from "@orderly.network/ui";

<Logo size="md" />
```

---

## DatePicker

Date selection component.

```tsx
import { DatePicker, DateRangePicker } from "@orderly.network/ui";

<DatePicker
  value={selectedDate}
  onChange={setSelectedDate}
  placeholder="Select date"
/>

<DateRangePicker
  value={dateRange}
  onChange={setDateRange}
/>
```

---

## Pagination

Pagination controls.

```tsx
import { PaginationItems } from "@orderly.network/ui";

<PaginationItems
  currentPage={page}
  totalPages={totalPages}
  onPageChange={setPage}
/>
```

---

# Domain-Specific Components

## Order Entry (`@orderly.network/ui-order-entry`)

### OrderEntry

Complete order entry form widget.

```tsx
import { OrderEntry, OrderEntryWidget, useOrderEntryScript } from "@orderly.network/ui-order-entry";

// Using Widget (recommended)
<OrderEntryWidget symbol="PERP_ETH_USDC" />

// Using Hook + UI separately
const orderEntryState = useOrderEntryScript({ symbol: "PERP_ETH_USDC" });
<OrderEntry {...orderEntryState} />
```

### OrderConfirmDialog

Order confirmation dialog.

```tsx
import { OrderConfirmDialog } from "@orderly.network/ui-order-entry";

<OrderConfirmDialog
  order={orderDetails}
  onConfirm={handleConfirm}
  onCancel={handleCancel}
/>
```

### AdditionalInfo

Order additional information display.

```tsx
import { AdditionalInfo } from "@orderly.network/ui-order-entry";

<AdditionalInfo
  estLiqPrice={12500}
  fees={0.05}
  maxQty={100}
/>
```

---

## Positions (`@orderly.network/ui-positions`)

### PositionsWidget

Complete positions management widget.

```tsx
import { PositionsWidget, MobilePositionsWidget } from "@orderly.network/ui-positions";

// Desktop
<PositionsWidget symbol="PERP_ETH_USDC" />

// Mobile
<MobilePositionsWidget />
```

### FundingFeeHistoryUI

Funding fee history display.

```tsx
import { FundingFeeHistoryUI, FundingFeeButton } from "@orderly.network/ui-positions";

<FundingFeeButton onClick={() => showFundingHistory()} />
<FundingFeeHistoryUI data={fundingHistory} />
```

---

## Orders (`@orderly.network/ui-orders`)

### OrdersWidget

Complete orders management widget.

```tsx
import { OrdersWidget, TabType } from "@orderly.network/ui-orders";

<OrdersWidget 
  symbol="PERP_ETH_USDC"
  defaultTab={TabType.PENDING}
/>
```

---

## Leverage (`@orderly.network/ui-leverage`)

### LeverageEditor

Leverage adjustment widget.

```tsx
import { LeverageEditor, LeverageSlider } from "@orderly.network/ui-leverage";

// Full editor widget
<LeverageEditor />

// Just the slider
<LeverageSlider
  value={leverage}
  onChange={setLeverage}
  max={20}
/>
```

### Show as Dialog/Sheet

```tsx
import { modal } from "@orderly.network/ui";
import { LeverageWidgetWithDialogId, LeverageWidgetWithSheetId } from "@orderly.network/ui-leverage";

// Open as dialog
modal.show(LeverageWidgetWithDialogId);

// Open as sheet (mobile)
modal.show(LeverageWidgetWithSheetId);
```

---

## Take-Profit/Stop-Loss (`@orderly.network/ui-tpsl`)

### PositionTPSLPopover

TP/SL popover for positions.

```tsx
import { PositionTPSLPopover, PositionTPSLSheet } from "@orderly.network/ui-tpsl";

// Desktop popover
<PositionTPSLPopover position={position} />

// Mobile sheet
<PositionTPSLSheet position={position} />
```

### TPSLAdvancedWidget

Advanced TP/SL configuration.

```tsx
import { TPSLAdvancedWidget } from "@orderly.network/ui-tpsl";

<TPSLAdvancedWidget
  symbol="PERP_ETH_USDC"
  side="BUY"
  onSubmit={handleTPSLSubmit}
/>
```

---

## Transfer (`@orderly.network/ui-transfer`)

### DepositForm / WithdrawForm

Deposit and withdrawal forms.

```tsx
import { 
  DepositForm, 
  WithdrawForm,
  DepositAndWithdraw,
  TransferForm 
} from "@orderly.network/ui-transfer";

// Separate forms
<DepositForm onDeposit={handleDeposit} />
<WithdrawForm onWithdraw={handleWithdraw} />

// Combined widget
<DepositAndWithdraw />

// Internal transfer
<TransferForm />
```

### Supporting Components

```tsx
import { 
  ChainSelect,
  QuantityInput,
  AvailableQuantity,
  Web3Wallet,
  BrokerWallet,
  ActionButton,
  Fee
} from "@orderly.network/ui-transfer";

<ChainSelect chains={chains} value={selectedChain} onChange={setChain} />
<QuantityInput value={amount} onChange={setAmount} max={maxAmount} />
<AvailableQuantity balance={balance} />
```

---

## Wallet Connector (`@orderly.network/ui-connector`)

### WalletConnectorWidget

Wallet connection widget.

```tsx
import { WalletConnectorWidget, WalletConnectorModalId } from "@orderly.network/ui-connector";
import { modal } from "@orderly.network/ui";

// Show connect modal
modal.show(WalletConnectorModalId);

// Or use component directly
<WalletConnectorWidget />
```

### AuthGuard

Authentication guard wrapper.

```tsx
import { AuthGuard, AuthGuardDataTable, AuthGuardEmpty } from "@orderly.network/ui-connector";

<AuthGuard fallback={<ConnectWalletPrompt />}>
  <ProtectedContent />
</AuthGuard>

// For DataTables
<AuthGuardDataTable>
  <DataTable ... />
</AuthGuardDataTable>
```

---

## Chain Selector (`@orderly.network/ui-chain-selector`)

### ChainSelectorWidget

Chain selection widget.

```tsx
import { ChainSelectorWidget, ChainSelectorDialogId } from "@orderly.network/ui-chain-selector";
import { modal } from "@orderly.network/ui";

// Show chain selector
modal.show(ChainSelectorDialogId);
```

---

## TradingView (`@orderly.network/ui-tradingview`)

### TradingviewWidget

TradingView chart integration.

```tsx
import { TradingviewWidget, TradingviewUI, useTradingviewScript } from "@orderly.network/ui-tradingview";

// Using widget
<TradingviewWidget 
  symbol="PERP_ETH_USDC"
  theme="dark"
/>

// Using hook + UI
const chartState = useTradingviewScript({ symbol: "PERP_ETH_USDC" });
<TradingviewUI {...chartState} />
```

---

## Share PnL (`@orderly.network/ui-share`)

### SharePnL

Share position PnL widget.

```tsx
import { SharePnLDialogId, SharePnLBottomSheetId } from "@orderly.network/ui-share";
import { modal } from "@orderly.network/ui";

// Show share dialog
modal.show(SharePnLDialogId, {
  position: positionData,
  pnl: 1500,
  pnlPercentage: 15.5,
});
```

---

## Scaffold (`@orderly.network/ui-scaffold`)

### Scaffold

Main app layout scaffold.

```tsx
import { 
  Scaffold, 
  MainNavWidget,
  BottomNavWidget,
  SideNavbarWidget,
  AccountMenuWidget
} from "@orderly.network/ui-scaffold";

<Scaffold
  header={<MainNavWidget />}
  sidebar={<SideNavbarWidget />}
  footer={<BottomNavWidget />}
>
  <MainContent />
</Scaffold>
```

### Navigation Components

```tsx
import { 
  MainNavWidget, 
  MainNavMobile,
  BottomNav,
  SideBar,
  AccountSummaryWidget,
  ChainMenuWidget
} from "@orderly.network/ui-scaffold";

<MainNavWidget 
  logo={<Logo />}
  trailing={<AccountMenuWidget />}
/>

<BottomNav items={navItems} />

<SideBar items={sideMenuItems} collapsed={isCollapsed} />
```

---

## Import Summary

### Base UI (`@orderly.network/ui`)

```tsx
import {
  // Layout
  Box, Flex, Grid,
  
  // Form Controls
  Button, Input, TextField, inputFormatter,
  Select, SelectItem,
  Checkbox, Switch, Slider,
  
  // Typography
  Text, Numeral,
  
  // Overlays
  Dialog, DialogContent, DialogHeader, DialogTitle, DialogBody, DialogFooter,
  Sheet, SheetContent, SheetHeader, SheetTitle,
  modal, ModalProvider,
  Popover, PopoverContent, PopoverTrigger,
  Tooltip,
  DropdownMenu, DropdownMenuContent, DropdownMenuItem,
  
  // Data Display
  DataTable, Tabs, TabsList, TabsTrigger, TabsContent,
  Card, CardHeader, CardContent, CardFooter,
  Badge, Divider,
  
  // Feedback
  toast, Toaster, Spinner, EmptyDataState,
  
  // Utility
  ScrollArea, Collapsible, Avatar, EVMAvatar, Logo,
  DatePicker, DateRangePicker, PaginationItems,
  
  // Theme
  OrderlyThemeProvider, useOrderlyTheme,
  
  // Utilities
  cn, tv,
} from "@orderly.network/ui";
```

### Domain Components

```tsx
// Order Entry
import { OrderEntry, OrderEntryWidget, useOrderEntryScript } from "@orderly.network/ui-order-entry";

// Positions
import { PositionsWidget, MobilePositionsWidget } from "@orderly.network/ui-positions";

// Orders
import { OrdersWidget, TabType } from "@orderly.network/ui-orders";

// Leverage
import { LeverageEditor, LeverageSlider } from "@orderly.network/ui-leverage";

// TP/SL
import { PositionTPSLPopover, TPSLAdvancedWidget } from "@orderly.network/ui-tpsl";

// Transfer
import { DepositForm, WithdrawForm, TransferForm } from "@orderly.network/ui-transfer";

// Connector
import { WalletConnectorWidget, AuthGuard } from "@orderly.network/ui-connector";

// Chain Selector
import { ChainSelectorWidget } from "@orderly.network/ui-chain-selector";

// TradingView
import { TradingviewWidget } from "@orderly.network/ui-tradingview";

// Share
import { SharePnLDialogId } from "@orderly.network/ui-share";

// Scaffold
import { Scaffold, MainNavWidget, BottomNavWidget } from "@orderly.network/ui-scaffold";
```

---

## Styling & Theming

### Tailwind CSS Integration

The UI library uses Tailwind CSS with the `oui-` prefix for all classes.

```tsx
import { OUITailwind } from "@orderly.network/ui";

// tailwind.config.js
module.exports = {
  presets: [OUITailwind.preset],
  content: [
    // ... your content paths
    "./node_modules/@orderly.network/ui/dist/**/*.{js,mjs}",
  ],
};
```

### Custom Theming

```tsx
import { OrderlyThemeProvider } from "@orderly.network/ui";

<OrderlyThemeProvider
  theme={{
    colors: {
      primary: "#your-primary-color",
      success: "#your-success-color",
      danger: "#your-danger-color",
    },
  }}
>
  <App />
</OrderlyThemeProvider>
```

### Using `cn` Utility

```tsx
import { cn } from "@orderly.network/ui";

<div className={cn(
  "oui-base-class",
  isActive && "oui-active-class",
  className
)} />
```

### Using `tv` (tailwind-variants)

```tsx
import { tv } from "@orderly.network/ui";

const buttonVariants = tv({
  base: "oui-px-4 oui-py-2 oui-rounded",
  variants: {
    color: {
      primary: "oui-bg-primary",
      secondary: "oui-bg-secondary",
    },
  },
});

<button className={buttonVariants({ color: "primary" })} />
```

---

## Best Practices

### 1. Use Widget Components for Complex Features

```tsx
// ✅ Good - Use pre-built widget
<OrderEntryWidget symbol="PERP_ETH_USDC" />

// ❌ Avoid - Building from scratch unless needed
<CustomOrderEntry />
```

### 2. Leverage the Modal System

```tsx
// ✅ Good - Use modal system for dialogs
modal.confirm({ title: "Confirm", content: "Are you sure?" });

// ❌ Avoid - Managing dialog state manually
const [isOpen, setIsOpen] = useState(false);
```

### 3. Use Responsive Components

```tsx
// Components that adapt to screen size
const { isMobile } = useScreen();

{isMobile ? (
  <MobilePositionsWidget />
) : (
  <PositionsWidget />
)}
```

### 4. Handle Loading States

```tsx
<DataTable
  data={data}
  loading={isLoading}
  emptyView={<EmptyDataState message="No data" />}
/>
```

### 5. Use AuthGuard for Protected Content

```tsx
<AuthGuard fallback={<ConnectPrompt />}>
  <ProtectedTradingUI />
</AuthGuard>
```

---

## Version Compatibility

- **UI Package**: v2.8.x
- **React**: ^18.0.0
- **Radix UI**: Various ^1.x versions
- **Tailwind CSS**: ^3.4.x
- **TanStack Table**: ^8.20.x

---

## Additional Resources

- [Orderly Network Documentation](https://orderly.network/docs)
- [Component Storybook](https://storybook.orderly.network)
- [SDK GitHub Repository](https://github.com/OrderlyNetwork/js-sdk)
