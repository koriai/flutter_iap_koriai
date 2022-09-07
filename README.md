# (작성중)

2022.09.07 작성중인 콘텐츠입니다.



Flutter 입앱 결제 모듈인 in_app_purchase 패키지 구현 내용입니다. 

get 패키지를 함께 사용하시는 분들에게 도움이 되었으면 좋겠습니다.



Koeran description of flutter package in_app_purchase, non-official


## Intro

패키지 : [in_app_purchase](https://pub.dev/packages/in_app_purchase)

* Other Examples

> [pub.dev/example](https://pub.dev/packages/in_app_purchase/example)
> 
> [구글 코드랩 flutter-in-app-purchases](https://codelabs.developers.google.com/codelabs/flutter-in-app-purchases#0)

* 도움이 되는 사이트들

> [codemagic/Understanding `in_app_purchase` APIs in Flutter](https://blog.codemagic.io/understanding-in-app-purchase-apis-in-flutter/)
>
>

* 개발환경

> Flutter version: 3.0.5
>
> Android version: 11


* 설치 - Installation

```flutter pub add get
flutter pub add in_app_purchase
```
or
````
dependencies:
    get: ^4.6.5
    in_app_purchase: ^3.0.7
````
## 설명

##### 0. 패키지 및 변수 선언

```
import 'package:flutter/material.dart';
import 'package:get/get.dart';
import 'package:in_app_purchase/in_app_purchase.dart';
import 'package:in_app_purchase_android/billing_client_wrappers.dart';
import 'package:in_app_purchase_android/in_app_purchase_android.dart';
import 'package:in_app_purchase_storekit/in_app_purchase_storekit.dart';
import 'package:cloud_functions/cloud_functions.dart';


class UserController extends GetxController {
    Rxn<List<ProductDetails>> sellingItems = Rxn<List<ProductDetails>>();
    Rxn<List<PurchaseDetails>> iapPurchaseDetails = Rxn<List<PurchaseDetails>>();
    Rxn<List<PurchaseReceipt>> receiptList = Rxn<List<PurchaseReceipt>>();
    Rxn<bool> hasActiveSubscription = Rxn<bool>();
    Rxn<bool> hasUpgrade = Rxn<bool>();
    Rxn<bool> storeAvailable = Rxn<bool>();
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


##### 1. 스토어 정보 불러오기

1. 플레이 콘솔에 등록된 스토어의 정보를 호출
> 

2. 현재 앱에서 판매할 아이템 리스트를 호출
> availableItems

```
availableItems = (List<String> 판매할 아이템 리스트).toSet();
_functions = FirebaseFunctions.instanceFor(region: CLOUD_REGION);
iap = InAppPurchase.instance;
iapPurchaseDetails.bindStream(iap.purchaseStream);
initPurchase();
```

3. iap 인스턴스가 사용가능한지 파악 
> await iap.isAvailable();

4. 아이템 리스트를 스토어에서 찾고, 아이템의 상세정보를 호출 & 저장
> await iap.queryProductDetails(availableItems)

```
Future<void> loadStore() async {
    storeAvailable.value = await iap.isAvailable();
    if (storeAvailable.value! == false) {
        return;
    }
    try {
        final ProductDetailsResponse productDetailResponse =
            await iap.queryProductDetails(availableItems);
        sellingItems.value = productDetailResponse.productDetails;
    } catch (e) {
        storeAvailable.value = false;
        return;
    }}
```

#### 2. 구매과정 구독

1. 구매 과정을 구독하는 인스턴스를 구성
2. 현재 플랫폼에 따라 **Android** or **iOS** 업데이트

```
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

```
void _listenToPurchaseUpdated(List<PurchaseDetails>? purchaseDetailsList) {
    if (purchaseDetailsList == null) return;
    purchaseDetailsList.forEach((PurchaseDetails purchaseDetails) async {
        <!-- purchaseStatus.value = purchaseDetails.status; -->
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
            print("pending complete");
            await iap.completePurchase(purchaseDetails);
        }
    });
}
```

#### 4. 결제 

```

  Future<void> handlePurchase(PurchaseDetails purchaseDetails) async {
    final validPurchase = await verifyPurchase(purchaseDetails);
    if (validPurchase) {
      try {
        if (availableConsumables.contains(purchaseDetails.productID)) {
          // await _deliverConsumable(purchaseDetails);
          print('deliver consumable');
        } else {
          // await _deliverSubscription(purchaseDetails);
          print('deliver subs');
        }

        Get.dialog(
          AlertDialog(
            title: Text("${purchaseDetails.productID} 구매 완료"),
            content: Text("구매해주셔서 감사합니다"),
          ),
        );
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


> import 'package:cloud_functions/cloud_functions.dart';
>
> const CLOUD_REGION = "asia-northeast3"
>
> _functions = FirebaseFunctions.instanceFor(region: CLOUD_REGION);


```
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

각 경우에 맞게 결제 후 진행하면 됩니다. (ex. alertDialog 등)

```
void _updateStreamOnDone() {
    // 결제 완료 알람
}

void handleInvalidPurchase(PurchaseDetails purchaseDetails) async {
    // 잘못된 결제 
}
void _handleError(PurchaseDetails purchaseDetails) async {
    // 결제 에러
}
```

#### 4. 결제 시작

결제하기를 누르고 결제되는 사항입니다.
**Android** 만 구현되었으며 **iOS**는 아래 코드 iOS 하에 입력하시면 됩니다.

```
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

// GooglePlayPurchaseDetails? _getOldSubscription(ProductDetails current) {
//   GooglePlayPurchaseDetails? oldSubscription;
//   final _list = receiptList.value
//       ?.where((e) => e.status == ProductStatus.active)
//       .toList();
//   // if (_list != null && _list.length > 0)
//   //   oldSubscription =
//   //       _list.first.purchaseDetails as GooglePlayPurchaseDetails;
//   return oldSubscription;
// }
```


## Firebase Cloud Function 구성

앱에서 결제를 한 다음, Firebase cloud function을 활용한 결제 확인 모듈입니다.

본 예에서는 TYPESCRIPT를 사용하였습니다.

아래 링크에서 보다 자세한 사항을 알아보실 수 있습니다.

> [구글 코드랩 flutter-in-app-purchases 10. Verify purchases](https://codelabs.developers.google.com/codelabs/flutter-in-app-purchases#9)

#### 1. 플랫폼 구분

```
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
```
export const expireSubscriptions = functions.runWith({
  minInstances: 1,
  timeoutSeconds: 300,
  memory: "512MB",
}).pubsub.schedule("every 5 mins")
    .onRun(() => iapRepository.expireSubscriptions());

```
