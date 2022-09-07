# Flutter 인앱 결제 제작기 (작성중)

2022.09.07 작성중인 콘텐츠입니다.

Flutter 입앱 결제 모듈인 in_app_purchase 패키지 구현 내용입니다.

get 패키지를 함께 사용하시는 분들에게 도움이 되었으면 좋겠습니다.

부분부분 필요에 맞게 수정하시며 됩니다.

Koeran description of flutter package in_app_purchase, non-official

## Intro

* 사용 패키지

> [in_app_purchase](https://pub.dev/packages/in_app_purchase)
>
> [get](https://pub.dev/packages/get)

* 참고 자료 및 예제

> [pub.dev/example](https://pub.dev/packages/in_app_purchase/example)
>
> [구글 코드랩 flutter-in-app-purchases (**추천)](https://codelabs.developers.google.com/codelabs/flutter-in-app-purchases#0)
>
> [codemagic/Understanding `in_app_purchase` APIs in Flutter](https://blog.codemagic.io/understanding-in-app-purchase-apis-in-flutter/)
>

* 개발환경

> Flutter version: 3.0.5
>
> Android version: 11

* 설치

```flutter
flutter pub add get
flutter pub add in_app_purchase
```

or

```flutter
dependencies:
    get: ^4.6.5
    in_app_purchase: ^3.0.7
```

## 설명

### Flutter 앱 내부

#### 0. 패키지 및 변수 선언

```flutter
import 'package:flutter/material.dart';
import 'package:get/get.dart';
import 'package:in_app_purchase/in_app_purchase.dart';
import 'package:in_app_purchase_android/billing_client_wrappers.dart';
import 'package:in_app_purchase_android/in_app_purchase_android.dart';
import 'package:in_app_purchase_storekit/in_app_purchase_storekit.dart';
import 'package:cloud_functions/cloud_functions.dart';


class UserController extends GetxController {
    // 판매 아이템 상세정보, 스토어에서 로드
    Rxn<List<ProductDetails>> sellingItems = 
    Rxn<List<ProductDetails>>();
    // 인앱결제 상세정보
    Rxn<List<PurchaseDetails>> iapPurchaseDetails = Rxn<List<PurchaseDetails>>();
    // 구매목록 (영수증)
    Rxn<List<PurchaseReceipt>> receiptList = Rxn<List<PurchaseReceipt>>();
    // 활성화된 구독 있는지 판단
    Rxn<bool> hasActiveSubscription = Rxn<bool>();
    Rxn<bool> hasUpgrade = Rxn<bool>();
    // 스토어 접속 가능 여부
    Rxn<bool> storeAvailable = Rxn<bool>();
    // 구매 상황 (구재중, 구매완료, pending)
    Rxn<PurchaseStatus> purchaseStatus = Rxn<PurchaseStatus>();

    InAppPurchaseAndroidPlatformAddition? androidAddition;
    InAppPurchaseStoreKitPlatformAddition? iosPlatformAddition;
    PurchaseDetails? latestSubscription;
    BillingResultWrapper? priceChangeConfirmationResult;
    
    late Set<String> availablItems;
    late InAppPurchase iap;
    late FirebaseFunctions _functions;
    late final Completer<bool> isInitialized = Completer();
    late FirebaseAuth _auth;
    
    }
```

#### 1. 스토어 정보 불러오기 (초기화)

* 플레이 콘솔에 등록된 스토어의 정보를 호출

* 현재 앱에서 판매할 아이템의 **이름** 리스트를 가져온다 (Remote Config 가능, 가격 등의 상세정보는 스토어에서 추후 로드)

* 각 인스턴스 초기화

>
> InAppPurchase.instance
>
> FirebaseFunctions.instanceFor(region: CLOUD_REGION) // Optaional
>

* 구매 Stream을 get의 구매 stream에 bind

```flutter
availableItems = (List<String> 판매할 아이템 리스트).toSet();
iap = InAppPurchase.instance;
iapPurchaseDetails.bindStream(iap.purchaseStream);
_functions = FirebaseFunctions.instanceFor(region: CLOUD_REGION);
initPurchase();
```

* iap 인스턴스가 사용가능한지 파악

> await iap.isAvailable();

* 아이템 리스트를 스토어에서 찾고, 아이템의 상세정보를 호출 & 저장

> await iap.queryProductDetails(availableItems)

```flutter
Future<void> loadStore() async {
    storeAvailable.value = await iap.isAvailable();
    if (storeAvailable.value! == false) {
        return;
    }
    try { // 아이템 쿼리를 가져오는데 실패하면 상점 이용불가
        final ProductDetailsResponse productDetailResponse =
            await iap.queryProductDetails(availableItems);
        sellingItems.value = productDetailResponse.productDetails;
    } catch (e) {
        storeAvailable.value = false;
        return;
    }}
```

#### 2. 구매과정 구독

* 구매 과정을 구독하는 인스턴스를 구성

> 구매 스트림: _listenToPurchaseUpdated
>
> onDone: 구매 후
>
> onError: 에러 발생시
> 

* 현재 플랫폼에 따라 **Android** or **iOS** 업데이트

```flutter
void initPurchase() async {
    iapPurchaseDetails.listen(
        _listenToPurchaseUpdated,
        onDone: _updateStreamOnDone,
        onError: _handleError,
    );
    if (Platform.isAndroid) {
        androidAddition = iap.getPlatformAddition<InAppPurchaseAndroidPlatformAddition>();
    } else if (Platform.isIOS) {
        iosPlatformAddition = iap.getPlatformAddition<InAppPurchaseStoreKitPlatformAddition>();
    }
}
```


#### 3. 구매과정 인스턴스 구성

> PurchaseDetails: in_app_purchase 에서

```flutter
void _listenToPurchaseUpdated(List<PurchaseDetails>? purchaseDetailsList) {
    if (purchaseDetailsList == null) return;
    purchaseDetailsList.forEach((PurchaseDetails purchaseDetails) async {
        if (purchaseDetails.status == PurchaseStatus.pending) {
            <!-- // purchaseStatus.value = purchaseDetails.status; -->
        } else {
            <!-- // purchaseStatus.value = purchaseDetails.status; -->
            if (purchaseDetails.status == PurchaseStatus.error) {
            } else if (purchaseDetails.status == PurchaseStatus.purchased ||
                    purchaseDetails.status == PurchaseStatus.restored) {
                try {
                    await handlePurchase(purchaseDetails);
                } catch (e) {
                    throw e;
                }
            }
        }

        if (purchaseDetails.pendingCompletePurchase) {
            await iap.completePurchase(purchaseDetails);
        }
    });
}
```

#### 4. 결제

결제 과정에서 서버와 연동 후 문제 없다면 해당 상품을 사용자에게 전달

> **validPurchase==True**

```flutter

  Future<void> handlePurchase(PurchaseDetails purchaseDetails) async {
    final validPurchase = await verifyPurchase(purchaseDetails);
    if (validPurchase) {
      try {
        if (availableConsumables.contains(purchaseDetails.productID)) {
          await _deliverConsumable(purchaseDetails);
        } else {
          await _deliverSubscription(purchaseDetails);
        }
        print("Thank you for purchasing");
      } catch (e) {
        print("error when deliver products $e");
      }
    } else
      handleInvalidPurchase(purchaseDetails);
    if (purchaseDetails.pendingCompletePurchase) {
      await iap.completePurchase(purchaseDetails);
    }
    if (Platform.isAndroid) {
      await androidAddition!.consumePurchase(purchaseDetails);
    }
    if (purchaseDetails.status == PurchaseStatus.error) {
      _handleError(purchaseDetails);
    }
  }
```

#### 5. 결제 확인

결제 확인을 위하여 파이어베이스 function을 사용합니다.
iap 결제 인스턴스가 구성될때 아래 firebasefunction 이 구성되어야합니다.
본문은 서울 리전(asia-northeast3)을 사용했습니다. 프로젝트에 맞게 설정하시면 됩니다.

> import 'package:cloud_functions/cloud_functions.dart';
>
> const CLOUD_REGION = "asia-northeast3"
>
> _functions = FirebaseFunctions.instanceFor(region: CLOUD_REGION);

결제 확인을 개별 서버에 진행하여도 되나, 본문에서는 firebase functions를 사용했습니다.

http Call로 해당 파라미터 전달 **source,verificationData,productId**

```flutter
Future<bool> verifyPurchase(PurchaseDetails purchaseDetails) async {
    final __functions = await firebaseFunctions;
    final HttpsCallableResult results =
        await __functions.httpsCallable('verifyPurchase')({
        'source': purchaseDetails.verificationData.source,
        'verificationData':
            purchaseDetails.verificationData.serverVerificationData,
        'productId': purchaseDetails.productID
    });
    return results.data as bool;
}
```

#### 4. 결제 시작

결제하기를 누르고 결제되는 사항입니다.
**Android** 만 구현되었으며 **iOS**는 아래 코드 iOS 하에 입력하시면 됩니다.

```flutter
Future<void> purchaseSubscription(productDetails,
    {List<PurchaseReceipt>? previous}) async {
    late PurchaseParam purchaseParam;
    if (Platform.isAndroid) {
        final _pastPurchases =
            (await androidAddition!.queryPastPurchases()).pastPurchases;
        if (_pastPurchases.isNotEmpty)
            purchaseParam = GooglePlayPurchaseParam(
                productDetails: productDetails,
                applicationUserName: null,
                changeSubscriptionParam: ChangeSubscriptionParam(
                oldPurchaseDetails: _pastPurchases.first,
                prorationMode: ProrationMode.immediateWithTimeProration,
                ));
        else
            purchaseParam = GooglePlayPurchaseParam(
                productDetails: productDetails,
                applicationUserName: null,
                changeSubscriptionParam: null);
    } else {
        // iOS
    }
    final _purchaseResult =
        await iap.buyNonConsumable(purchaseParam: purchaseParam);
    print('PurchaseResult of Subscription: $_purchaseResult');
}

// Button response of consumable
Future<void> purchaseConsumable(productDetails) async {
    PurchaseParam purchaseParam = GooglePlayPurchaseParam(
        productDetails: productDetails,
        applicationUserName: null,
        changeSubscriptionParam: null);
    final _purchaseResult = await iap.buyConsumable(
        purchaseParam: purchaseParam, autoConsume: true);
    print('PurchaseResult of buyConsumable: $_purchaseResult');
}

Future<void> restoreUserPurchases() async {
    if (Platform.isAndroid) {
    final _pastPurchases = await androidAddition!.queryPastPurchases();

    _pastPurchases.pastPurchases.forEach((past) async {
        if (past.pendingCompletePurchase == true) {
        await iap.restorePurchases(
            applicationUserName: firebaseUser.value?.uid);
        // await iap.completePurchase(past);
        }
    });
    }
}
```

## Firebase Cloud Function 구성 (TypeScript)

앱에서 결제를 한 다음, Firebase cloud function을 활용한 결제 확인 모듈입니다.

본 예에서는 TYPESCRIPT를 사용하였습니다.

아래 링크에서 보다 자세한 사항을 알아보실 수 있습니다.

> [구글 코드랩 flutter-in-app-purchases 10. Verify purchases](https://codelabs.developers.google.com/codelabs/flutter-in-app-purchases#9)

### 1. 플랫폼 구분

1. https.onCall 로 전달받은 인자를 토대로 구매 정보가 유효한지 검증

2. 구매 정보에 따라 플랫폼 별 메소드를 호출

> Android) google-play.purchase-handler.ts
>
> iOS) app-store.purchase-handler.ts

```typescript
const iapRepository = new IapRepository(admin.firestore());
const purchaseHandlers: { [source in IAPSource]: PurchaseHandler } = {
  "google_play": new GooglePlayPurchaseHandler(iapRepository),
  "app_store": new AppStorePurchaseHandler(iapRepository),
};

interface VerifyPurchaseParams {
  source: IAPSource;
  verificationData: string;
  productId: string;
}

export const verifyPurchase = functions.https.onCall(
    async (
        data: VerifyPurchaseParams,
        context,
    ): Promise<boolean> => {
      // Check authentication
      if (!context.auth) {
        console.warn("verifyPurchase called when not authenticated");
        throw new HttpsError(
            "unauthenticated",
            "Request was not authenticated.",
        );
      }
      // Get the product data from the map
      const productData = productDataMap[data.productId];
      // If it was for an unknown product, do not process it.
      if (!productData) {
        console.warn(`verifyPurchase called for an unknown product ("${data.productId}")`);
        return false;
      }
      // If it was for an unknown source, do not process it.
      if (!purchaseHandlers[data.source]) {
        console.warn(`verifyPurchase called for an unknown source ("${data.source}")`);
        return false;
      }
      // Process the purchase for the product
      return purchaseHandlers[data.source].verifyPurchase(
          context.auth.uid,
          productData,
          data.verificationData,
      );
    });
```

구독을 할 경우 스토어에서 구독 신청이 되기 때문에
아래처럼 **handleServerEvent**를 별도로 처리하여야 한다.

```typescript
functions.pubsub.topic(GOOGLE_PLAY_PUBSUB_BILLING_TOPIC)
      .onPublish(async (message) => {})
```

GOOGLE_PLAY_PUBSUB_BILLING_TOPIC 의 경우, 플레이 콘솔에서 구글 프로젝트에서 연동하여야 한다.

> [구글 코드랩 flutter-in-app-purchases 11. Keep track of purchases](https://codelabs.developers.google.com/codelabs/flutter-in-app-purchases#10)

```typescript
// Handling of AppStore server-to-server events
export const handleAppStoreServerEvent =
    (purchaseHandlers.app_store as AppStorePurchaseHandler)
        .handleServerEvent;

// Handling of PlayStore server-to-server events
export const handlePlayStoreServerEvent =
    (purchaseHandlers.google_play as GooglePlayPurchaseHandler)
        .handleServerEvent;
```

#### 구독 만료의 경우

사용자가 버튼을 눌러주는 것이 아니기 때문에 **functions.https.onCall**으로 구현하지 말고, **functions.runWith.pubsub.schedule**을 구성하면 됩니다.

매 5분간 호출 &
최소 1개의 인스턴스 유지 & 메모리 512MB 사용

```typescript
export const expireSubscriptions = functions.runWith({
  minInstances: 1,
  timeoutSeconds: 300,
  memory: "512MB",
}).pubsub.schedule("every 5 mins")
    .onRun(() => iapRepository.expireSubscriptions());

```
