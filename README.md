iap_validation
================

*** THIS CODE IS ONLY NEEDED IN iOS 5.x and PRIOR. The vulnerability was fixed in iOS 6. ***

In July 2012, a Russian hacker identified a vulnerability in the In-App Purchase system and released it into the wild. Apple responded with TP40012484, "In-App Purchase Receipt Validation on iOS," which included VerificationController, a mostly-complete singleton class to perform validation.

Apple's VerificationController requires making a number of changes within the code and  adding your own callbacks and base64 implementation. This small collection of classes is based on it and provides a simple delegate-based system to perform validation. 

It's in the public domain with USE_CODE_REQUIRING_ATTRIBUTION not defined in RRBase64Manager. If it *is* defined, which is the default, a more efficient base64 implementation which does require attribution will be used; see that file for details.

iOS 5.x is required.  On iOS 4, all purchases will validate (just as if you weren't using the class at all).  Also, but sure to add linker flags "-fobjc-arc -weak-lSystem.B -weaklobjc" to avoid crashing with ARC enabled for the file when run on iOS 4. 

The iOS 5 requirement is only because of the use of NSJSONSerialization; patches to replace NSJSONSerialization with an iOS 4 compatibility implementation (in NSData+RRTransactionParsingAdditions) are welcome.

The verification code tries first with the deployment verification servers, and retries with the sandbox if the returned error code indicates this is needed. Therefore, no changes are needed between debug and deployment builds.

Use
--------

This example assumes that your `SKPaymentTransactionObserver` conformant class will also implement `RRVerificationControllerDelegate`.

You'll need your iTunes Connect In App Purchase Shared Secret, which you can find/generate via **iTunes Connect -> Manage Apps -> your app -> Manage In App Purchases**.

```objective-c
 #define MY_SHARED_SECRET	@"1234567890abcdef1234567890abcdef"
 
- (void)paymentQueue:(id)queue updatedTransactions:(NSArray *)transactions
{
	[RRVerificationController sharedInstance].itcContentProviderSharedSecret = MY_SHARED_SECRET;
 
	for (SKPaymentTransaction *transaction in transactions)
	{
		switch ([transaction transactionState])
		{
			case SKPaymentTransactionStatePurchasing:
				break;
			case SKPaymentTransactionStatePurchased:
			     /* If verification is successful, the delegate's verificationControllerDidVerifyPurchase:isValid: method will be called to take appropriate action and complete the transaction */
				if ([[RRVerificationController sharedInstance] verifyPurchase:transaction
																 withDelegate:self
																		error:NULL] == FALSE) {
					[self failedTransaction:transaction];
				}
				break;
			case SKPaymentTransactionStateFailed:
				[self failedTransaction:transaction];
				break;
			case SKPaymentTransactionStateRestored:
				/* If verification is successful, the delegate's verificationControllerDidVerifyPurchase:isValid: method will be called to take appropriate action and complete the transaction */
				if ([[RRVerificationController sharedInstance] verifyPurchase:transaction
																 withDelegate:self
																		error:NULL] == FALSE) {
					[self failedTransaction:transaction];
				}
			default:
				break;
		}
	}
}
```

RRVerificationControllerDelegate is implemented like so:

```objective-c
- (void)completeTransaction:(SKPaymentTransaction *)transaction 
{
 /* Handled a completed and verified new transcation */
}

- (void)restoreTransaction:(SKPaymentTransaction *)transaction 
{
 /* Handled a completed and verified restored transcation.
  * For many use cases this is simply [self completeTransaction:transaction]
  */
}

/*!
 * @brief Verification with Apple's server completed successfully
 *
 * @param transaction The transaction being verified
 * @param isValid YES if Apple reported the transaction was valid; NO if Apple said it was not valid or if the server's validation reply was inconsistent with validity
 */
 - (void)verificationControllerDidVerifyPurchase:(SKPaymentTransaction *)transaction isValid:(BOOL)isValid
 {
 	if (isValid) {
       switch (transaction.transactionState) {
            case SKPaymentTransactionStatePurchased:
                [self completeTransaction:transaction];
                break;
            case SKPaymentTransactionStateRestored:
                [self restoreTransaction:transaction];
            default:
                break;
        }

    } else
		[self displayFailureMessage];
		
	if (transaction.transactionState != SKPaymentTransactionStatePurchasing)
		[[SKPaymentQueue defaultQueue] finishTransaction:transaction];
 }
 
 /*!
  * @brief The attempt at verification could not be completed
  *
  * This does not mean that Apple reported the transaction was invalid, but
  * rather indicates a communication failure, a server error, or the like.
  *
  * @param transaction The transaction being verified
  * @param error An NSError describing the error. May be nil if the cause of the error was unknown (or if nobody has written code to report an NSError for that failure...)
  */
 - (void)verificationControllerDidFailToVerifyPurchase:(SKPaymentTransaction *)transaction error:(NSError *)error
 {
	NSString *message = NSLocalizedString(@"Your purchase could not be verified with Apple's servers. Please try again later.", nil);
	if (error) {
		message = [message stringByAppendingString:@"\n\n"];
		message = [message stringByAppendingFormat:NSLocalizedString(@"The error was: %@.", nil), error.localizedDescription];
	}
 
	[[[UIAlertView alloc] initWithTitle:NSLocalizedString(@"Purchase Verification Failed", nil)
		message:message
		delegate:nil
		cancelButtonTitle:NSLocalizedString(@"Dismiss", nil)
		otherButtonTitles:nil] show];
 }
```
Everything else is handled automatically; just put your own logic in performUpgrade, failedTransaction:, and displayFailureMessage, with no changes needed to the verification controller code.

These classes are intended to be compiled with ARC enabled. Be sure to specify `-fobjc-arc` in the 'Compile Sources' Build Phase for each file if you aren't using ARC project-wide 

You'll need to link against Security.framework to use the classes.
