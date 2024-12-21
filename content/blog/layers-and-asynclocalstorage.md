+++
author =" Mahamed Belkheir"
title = "AsyncLocalStorage and Layered Applications in Nodejs"
date = "2024-12-21"
description = "Dealing with cross-cutting concerns in a layered architecture using AsyncLocalStorage"
draft = false
+++

When reading through code, it's easier to understand when there's only one thing going on, one goal or concern to worry about, that to me, is the main benefit of layered architecture.

As code bases grow, having one layer to your code doing everything quickly grows uncontrolable, so I've eventually come to divide my API codebases to the following layers:

- Controllers (Representation layer)
- Usecases (Business logic layer) 
- Repositories (Data persistence layer)

```ts
// an example for the representation layer of an API
router.post( // define route method
    "/make-transaction", // and route path
    authenticationMiddleware, // authenticate the user credentials 
    async (req, res) => 
        {
            // validate payload coming from outside the system
            const payload = MakeTransactionSchema.validate(req.body);
            // offload any thinking to the business layer
            const result = await businessLayer.makeTransaction(req.user, payload);
            // control response structure
            res.json({
                status: "success",
                data: result
            });
        }
)
```
So in the previous example we defined a representation layer concerned about how our API is used, rather than what it actually does, if we are required to make changes to allow users to pay transactions even if they don't have money, we know we won't be passing by the routers.

```ts
// a (terrible) example of a bank's business layer

// defining what the input for the layer is
async function makeTransaction(user: User, transactionRequest: TransactionRequest) {
    // fetching relevant data from the data layer without worrying about how it's done
    const userBankAccount = await db.bankAccount.findOneOrFail({
        ownerId: user.id, 
        accountNumber: transactionRequest.fromAccountNumber
        })
    const targetBankAccount = await db.bankAccount.findOneOrFail({
        accountNumber: transactionRequest.toAccountNumber
    });

    // simplified business logic for an internal bank transfer
    userBankAccount.credit -= transactionRequest.amount;
    if (userBankAccount.credit < userBankAccount.overDraftLimit) {
        throw new InsufficientFundsError();
    }
    targetBankAccount.credit += transactionRequest.amount;
    
    // save changes back to data persistence layer, without worrying how exactly
    await db.bankAccount.save(userBankAccount);
    await db.bankAccount.save(targetBankAccount);

    // return relevant data to representation layer, in this case the new balance
    return {
        accountNumber: userBankAccount.accountNumber,
        newCreditAmount: userBankAccount.credit
    }
}
/**
 * A real banking backend would use double entry booking and would credit/debit amounts on accounts, instead of directly manipulating the values. and would run fraud detection, compliance checks, etc before commiting
 */
```

In the above example we defined a bank transfer's business layer, where we do not care how the API request arrives, or how the database is interacted with, our persistence could be an SQL DB, a DocumentDB or another service we call out to.

For the data persistence layer, an ORM or a class with SQL queries would suffice for now, we will explore a need for a custom one later in this post.

Now one problem with our approach is transactions, the business layer should not be aware of data layer specifics as much as possible, but transactions are important enough to break this consideration.

One approach is to let the business layer start a transaction and pass the transaction reference to each data layer call

```ts
async function makeTransaction(user: User, transactionRequest: TransactionRequest) {
    const trx = await db.startTransaction();
    const userBankAccount = await db.bankAccount.findOneOrFail({
        ownerId: user.id, 
        accountNumber: transactionRequest.fromAccountNumber
        }, trx)
    const targetBankAccount = await db.bankAccount.findOneOrFail({
        accountNumber: transactionRequest.toAccountNumber
    }, trx);

    userBankAccount.credit -= transactionRequest.amount;
    if (userBankAccount.credit < userBankAccount.overDraftLimit) {
        throw new InsufficientFundsError();
    }
    targetBankAccount.credit += transactionRequest.amount;
    
    await db.bankAccount.save(userBankAccount, trx);
    await db.bankAccount.save(targetBankAccount, trx);

    await trx.commit()
    return {
        accountNumber: userBankAccount.accountNumber,
        newCreditAmount: userBankAccount.credit
    }
}
```

This approach works, but it has some consequences:

- You must tediously pass around the transaction reference, and every function from the database layer must accomodate it as a parameter
- if your data layer allows operating without transactions (e.g. trx is an optional paramater), it can be easy to miss a call, having parts of your queries operate outside of your transaction can be disasterous
- if you start abstracting your business layer into more functions, you'll need to pass the transaction around to other service functions

And generally, aside from starting and ending the transaction, the business layer has NO interactions with the transaction itself, aside from passing it around, being given an option where the only answer you're supposed to say is "Yes" is not an "option" I want to have.

One way to go about sharing cross cutting concerns like transactions is [AsyncLocalStorage](https://nodejs.org/api/async_context.html#class-asynclocalstorage), in short, it offers you a way to set a value, go down into multiple calls down your callstack, and retrieve that same value. It solves the problem where global values are shared between requests in Nodejs, ASL values can be considered "globals" inside one request.

```ts
export const trxStorage = new AsyncLocalStorage<Transaction>();

export const startTrx = async <T>(cb: () => T ): T => {
    const trx = await db.startTransaction();
    const result = await trxStorage.start(trx, cb);
    await trx.commit();
    return result;
}
```
Here we setup the ASL instance, and provide a helper to start transactions, set the async local storage context, and commit the transaction before returning the result, we can now refactor our business layer to make use of this.


```ts
async function makeTransaction(user: User, transactionRequest: TransactionRequest) {
    return startTrx(async () => {
        // anything inside this callback is now able to use trxStorage.getStore() to access the same transaction object
        const userBankAccount = await db.bankAccount.findOneOrFail({
            ownerId: user.id, 
            accountNumber: transactionRequest.fromAccountNumber
            })
        const targetBankAccount = await db.bankAccount.findOneOrFail({
            accountNumber: transactionRequest.toAccountNumber
        });

        userBankAccount.credit -= transactionRequest.amount;
        if (userBankAccount.credit < userBankAccount.overDraftLimit) {
            throw new InsufficientFundsError();
        }
        targetBankAccount.credit += transactionRequest.amount;
        
        await db.bankAccount.save(userBankAccount);
        await db.bankAccount.save(targetBankAccount);

        return {
            accountNumber: userBankAccount.accountNumber,
            newCreditAmount: userBankAccount.credit
        }

    })
}
```
In this approach we no longer fear missing out on passing the transaction to the repository layer, and can share the transaction transparently between different functions without changing the parameters.

But we're not done yet, most nodejs ORMs, query builder and db clients do not support ASL out of the box for transactions, thus a need to wrap them arises, this is where a repository layer pays off.

```ts
export class BankAccountRepository {
    protected q() {
        const trx = trxStorage.getStore();
        const query = db.queryBuilder();
        if (trx) {
            query.transacting(trx);
        }
        return query;
    }

    async findOneOrFail(bankAccountDetails: Partial<BankAccountNumber>) {
        const data = await this.q().where(bankAccountDetails).first();
        if (!data) {
            throw new BankAccountNotFoundError();
        }
        return data;
    }
}
```

With this, we're able to start to share transactions across our codebase transparently, while centralizing how to start and handle transactions,
we're able to modify the startTrx function and add more error handling, for example, re-trying transactions when they fail due to a race condition.

for a more fleshed out example, you can checkout my [nodejs api template](https://github.com/Mahamed-Belkheir/nodejs-api-template), which is very opinionated, but it show cases the following examples:
- [Using the transaction starter utility](https://github.com/Mahamed-Belkheir/nodejs-api-template/blob/main/src/usecases/auth/user.ts#L19)
- [The transaction context](https://github.com/Mahamed-Belkheir/nodejs-api-template/blob/main/src/db/trx.ts)
- [The base repository using the transaction](https://github.com/Mahamed-Belkheir/nodejs-api-template/blob/main/src/db/repository.ts)

---

Transaction tracking is just one usecase, a lot of cross cutting concerns are annoying to deal with traditionally, two usecases I've found useful are multi-tenancy tracking, in the same vein as tracking the current transaction and setting queries to use it, we are able to check the current user's tenant and scope queries to it automatically.

We're also able to set request specific information, like traceIds, and log them everywhere inside the application, without having to pass around a request context god object including everything going on currently with the request.