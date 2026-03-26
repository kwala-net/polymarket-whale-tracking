# Polymarket – Whale Tracking

## Overview

Large traders (“whales”) on prediction markets often move markets quickly when reacting to news or private information. Tracking their activity can provide useful signals for research, alerts, or automated trading strategies.

In this guide we build a **Kwala workflow that tracks trades from a specific Polymarket address**. Whenever that address executes a trade, the system emits a filtered event and triggers a Telegram alert.

The system consists of three workflows:

1. Deploy a **Kwala function** that filters events.
2. Listen to **Polymarket `OrdersMatched` events** and pass them to the function.
3. Trigger an **alert workflow** when a matching trade is detected.

This pattern demonstrates how Kwala can be used to **monitor on-chain activity and trigger external actions in real time**.

---

## Complete end-to-end workflow configuration

This section presents a complete setup using three workflows. Together, they create a **Polymarket address tracker**.

| Workflow   | Purpose                   |
| ---------- | ------------------------- |
| Workflow 1 | Deploy filtering logic    |
| Workflow 2 | Monitor Polymarket trades |
| Workflow 3 | Trigger alerts            |

---

## Workflow 1: Deploy Kwala function to filter whale orders

The first workflow deploys a **Kwala Go function** to the Kalp chain.

This function receives parameters from Polymarket trade events and checks whether the trade was executed by the whale address we want to track. If it matches, it emits a `FilteredOrdersMatched` event.

**Go contract logic (decoded):**

```go theme={null}
type OrdersMatched struct {
	TakerOrderHash    string `json:"takerOrderHash"`
	TakerOrderMaker   string `json:"takerOrderMaker"`
	MakerAssetId      uint64 `json:"makerAssetId"`
	TakerAssetId      uint64 `json:"takerAssetId"`
	MakerAmountFilled uint64 `json:"makerAmountFilled"`
	TakerAmountFilled uint64 `json:"takerAmountFilled"`
}

func ComputeFilteredOrders(takerOrderHash string, takerOrderMaker string, makerAssetId uint64, takerAssetId uint64, makerAmountFilled uint64, takerAmountFilled uint64) error {
	// Emit event only for high-value transfers
	if takerOrderMaker == "<polymarket-whale-address>" {
		eventData := OrdersMatched{
			TakerOrderHash:    takerOrderHash,
			TakerOrderMaker:   takerOrderMaker,
			MakerAssetId:      makerAssetId,
			TakerAssetId:      takerAssetId,
			MakerAmountFilled: makerAmountFilled,
			TakerAmountFilled: takerAmountFilled,
		}

		eventPayload, _ := json.Marshal(eventData)
		EmitEvent("FilteredOrdersMatched", eventPayload)
	}
	return nil
}
```

**YAML 1: Deployment kwala function**

```yaml theme={null}
Name: deploycontract-event_filter
Trigger:
  TriggerSourceContract: NA
  TriggerChainID: NA
  TriggerEventName: NA
  RepeatEvery: NA
  ExecuteAfter: immediate
  ExpiresIn: 1865620000
Actions:
  - Name: Deploy
    Type: deploy
    ChainID: 1905
    EncodedGoContract: <base64-encoded-go-contract>
    RetriesUntilSuccess: 5
Execution:
  Mode: parallel
```

After this workflow runs, the deployed function can be called by other workflows.

---

## Workflow 2: Monitor Polymarket trades

This workflow listens for `OrdersMatched` events emitted by the **Polymarket CTFExchange contract** on Polygon.

Each event represents a trade on Polymarket. The workflow forwards the event parameters to the deployed Kwala function (`ComputeFilteredOrders`).

If the trader address matches the whale address, the function emits a `FilteredOrdersMatched` event.

**YAML 2: Address tracking workflow**

```yaml theme={null}
Name: filter_whale_orders_polymarket
Trigger:
  TriggerSourceContract: 0x4bFb41d5B3570DeFd03C39a9A4D8dE6Bd8B8982E
  TriggerChainID: 137
  TriggerEventName: OrdersMatched(bytes32,address,uint256,uint256,uint256,uint256)
  TriggerEventFilter: NA
  TriggerSourceContractABI: <source-contract-abi>
  TriggerPrice: NA
  RecurringSourceContract: 0x4bFb41d5B3570DeFd03C39a9A4D8dE6Bd8B8982E
  RecurringChainID: 137
  RecurringEventName: OrdersMatched(bytes32,address,uint256,uint256,uint256,uint256)
  RecurringEventFilter: NA
  RecurringSourceContractABI: <source-contract-abi>
  RecurringPrice: NA
  RepeatEvery: event
  ExecuteAfter: event
  ExpiresIn: 1794395460
  Meta: NA
  ActionStatusNotificationPOSTURL:
  ActionStatusNotificationAPIKey: NA
Actions:
  - Name: filter_ctf_orders
    Type: call
    SmartWallet: DEFAULT_SMART_WALLET
    APIEndpoint: NA
    APIPayload:
      Message: NA
    TargetContract: 7e63825c387746eaa1d2b4151e84580ccc4cf3de
    TargetFunction: func ComputeFilteredOrders()
    TargetParams:
      - re.event(0)
      - re.event(1)
      - re.event(2)
      - re.event(3)
      - re.event(4)
      - re.event(5)
    ChainID: 1906
    EncodedABI: NA
    Bytecode: NA
    EncodedGoContract: NA
    Metadata: NA
    RetriesUntilSuccess: 5
Execution:
  Mode: parallel
```

At this stage, the system continuously watches Polymarket trades and emits events whenever the tracked address executes an order.

---

## Workflow 3: Telegram alert workflow

The final workflow listens for the `FilteredOrdersMatched` event emitted by the Kwala function.

When the whale executes a trade, the workflow sends a **Telegram notification** with the trade details.

**YAML 3: Telegram alert workflow**

```yaml theme={null}
Name: event_listen_and_notify_tg
Trigger:
  TriggerSourceContract: 7e63825c387746eaa1d2b4151e84580ccc4cf3de
  TriggerChainID: 1905
  TriggerEventName: FilteredOrdersMatched(takerOrderHash string, takerOrderMaker string, makerAssetId uint64, takerAssetId uint64, makerAmountFilled uint64, takerAmountFilled uint64)
  RecurringSourceContract: 7e63825c387746eaa1d2b4151e84580ccc4cf3de
  RecurringChainID: 1905
  RecurringEventName: FilteredOrdersMatched(takerOrderHash string, takerOrderMaker string, makerAssetId uint64, takerAssetId uint64, makerAmountFilled uint64, takerAmountFilled uint64)
  RepeatEvery: event
  ExecuteAfter: event
  ExpiresIn: 1765558800
Actions:
  - Name: TELEGRAM_BOAT
    Type: post
    APIEndpoint: https://api.telegram.org/bot<your-bot-token>/sendMessage
    APIPayload:
      chat_id: "<your-chat-id>"
      text: re.event(1)'s polymarket order of re.event(4) tokens of tokenId re.event(2) was matched
    RetriesUntilSuccess: 5
Execution:
  Mode: parallel
```
