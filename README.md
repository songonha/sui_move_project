# sui_move_project
sui-move-project
1. Basic Types of Object
1.1. Object by Owner & Object by Object
1.1.1. In Code
module eamon::EAMON {
    use sui:tx_context::{Self, TxContext};
    use sui:transfer::transfer;

    fun init(ctx: &mut TxContext) {
        transfer::transfer({object}, tx_context::signer(ctx));
    }
}
1.1.2. Transfer Owner Objects
public fun transfer<T: key>(obj: T, recipient: address) {
    // assert ----
    // abort ----
    transfer_impl(obj, recipient)
}
1.1.3. impl Transfer Owner Objects
spec transfer_impl {
    pragma opaque;
    // never aborts as it requires object by-value and:
    // - it's OK to transfer whether object is fresh or already owned
    // - shared or immutable object cannot be passed by value
    aborts_if [abstract] false;
    modifies [abstract] global<object::Ownership>(object::id(obj).bytes);
    ensures [abstract] exists<object::Ownership>(object::id(obj).bytes);
    ensures [abstract] global<object::Ownership>(object::id(obj).bytes).owner == recipient;
    ensures [abstract] global<object::Ownership>(object::id(obj).bytes).status == prover::OWNED;
}
1.2. Shared Object
1.2.1. In Code
module eamon::EAMON {
    use sui:tx_context::{Self, TxContext};
    use sui:transfer::transfer;

    fun init(ctx: &mut TxContext) {
        transfer::share_object({object}, tx_context::signer(ctx));
    }
}
1.2.2. Transfer Shared Objects
public fun share_object<T: key>(obj: T) {
    share_object_impl(obj)
}
1.2.3. impl Transfer Shared Objects
spec share_object_impl {
    pragma opaque;
    aborts_if [abstract] sui::prover::owned(obj);
    modifies [abstract] global<object::Ownership>(object::id(obj).bytes);
    ensures [abstract] exists<object::Ownership>(object::id(obj).bytes);
    ensures [abstract] global<object::Ownership>(object::id(obj).bytes).status == prover::SHARED;
}
1.3. Immutable Object
1.3.1. In Code
module eamon::EAMON {
    use sui:tx_context::{Self, TxContext};
    use sui:transfer::transfer;

    fun init(ctx: &mut TxContext) {
        transfer::freeze_object({object}, tx_context::signer(ctx));
    }
}
1.3.2. Transfer Immutable Objects
public fun freeze_object<T: key>(obj: T) {
    freeze_object_impl(obj)
}
1.3.3. Impl Transfer Immutable Objects
spec freeze_object_impl {
    pragma opaque;
    // never aborts as it requires object by-value and:
    // - it's OK to freeze whether object is fresh or owned
    // - shared or immutable object cannot be passed by value
    aborts_if [abstract] false;
    modifies [abstract] global<object::Ownership>(object::id(obj).bytes);
    ensures [abstract] exists<object::Ownership>(object::id(obj).bytes);
    ensures [abstract] global<object::Ownership>(object::id(obj).bytes).status == prover::IMMUTABLE;
}
2. Ability
Abilities are keywords in Sui Move that define how types behave at the compiler level.

Abilities are crucial to defining how object behave in Sui Move at the language level. Each unique combination of abilities in Sui Move is its own design pattern. We will study abitilies and how to use them in Sui Move throughout the course.

Copy: value can be copied (or cloned by value)
Drop: value can be dropped by the end of scope
Key: value can be used as a key for global storage operations
Store: value can be stored inside global storage
Custom types that have the abilities key and store are considered to be assets in Sui Move. Assets are stored in global storage and can be transferred between accounts.

module eamon::EAMON {
    use sui::object::{Self, UID};

    struct Wallet has key, store, copy, drop {
        id: UID,
    }
}
3. CRUD Features
3.1. Create
module eamon::EAMON {
    use sui::object::{Self, UID};
    use sui::tx_context::{Self,TxContext}

    struct Wallet has key, store {
        id: UID,
        name: vector<u8>,
        balance: u128,
    }

    struct User has key {
        id: UID,
        name: String,
        wallet: Wallet,
    }

    fun init(ctx: &mut TxContext) {
        let wallet = Wallet {
            id: object::new(ctx),
            name: b"Eamon",
            balance: 10000000000
        };
        transfer::transfer(wallet, tx_context::signer(ctx));
    }
}
3.2. Read
public entry fun get_wallet(wallet: &Wallet): (u128, vector<u8>) {
    (wallet.balance, wallet.name)
}
3.3. Update
public entry fun update_balance(wallet: &mut Wallet, balance: u128) {
    wallet.balance = balance;
}
3.4. Delete
public entry fun delete_wallet(wallet: Wallet) {
    let Wallet {id, balance:_, name: _} = wallet;
    object::delete(id);
}
3.5. Change type of Ownership
public fun change_to_share_object(wallet: Wallet) {
    transfer::share_object(wallet);
}

public fun change_to_freeze_object(wallet: Wallet) {
    transfer::freeze_object(wallet);
}
4. Flexible Objects
4.1. Wrapping object
module eamon::EAMON {
    use sui::dynamic_object_field as ofield;
    use sui::object::{Self, UID};
    use sui::tx_context::{Self,TxContext}
    use sui::transfer;

    struct Bank has key {
        id: UID,
        wallets_created: u64,
        users_created: u64,
    }

    struct Wallet has key, store {
        id: UID,
        symbol: vector<u8>,
        balance: u64,
    }

    struct User has key, store {
        id: UID,
        wallet: Wallet,
        intended_address: address,
    }

    fun init(ctx: &mut TxContext) {
        let bank = Bank{
            id: object::new(ctx),
            wallets_created: 0,
            users_created: 0,
        };

        transfer::transfer(bank, tx_context::sender(ctx));
    }

    public entry fun create_wallet(bank: &mut Bank,symbol: vector<u8>, balance: u64, ctx: &mut TxContext) {

        let wallet = Wallet{id: object::new(ctx),
            symbol,
            balance,
        };
        bank.wallets_created = bank.wallets_created + 1;
        transfer::transfer(wallet, tx_context::sender(ctx));
    }
}
4.2. Wrap
public entry fun request_wallet(bank: &mut Bank, intended_address: address, wallet: Wallet, ctx: &mut TxContext) {
    let user = User{
        id: object::new(ctx),
        wallet,
        intended_address,
    };
    bank.users_created = bank.users_created + 1;
    transfer::transfer(user, intended_address);
}
4.3. Unwrap
public entry fun unpack_wallet(user: User, ctx: &mut TxContext) {
    let User {
        id,
        wallet,
        intended_address: _,
    } = us
        transfer::transfer(wallet, tx_context::sender(ctx));
    object::delete(id);
}
5. Dynamic Objects
5.1. Dynamic Object Field
use sui::dynamic_object_field as ofield;

public entry fun mutable_symbol_wallet(wallet: &mut Wallet, symbol: vector<u8>) {
    wallet.symbol = symbol;
}

public fun add_new_wallet(user: &mut User, wallet: Wallet, name: vector<u8>) {
    ofield::add(&mut user.id, name, wallet);
}
5.2. Mutate Dynamic Object
public entry fun mutate_symbol_wallet_via_user(user: &mut User, wallet_name: vector<u8>, symbol: vector<u8>) {
    mutable_symbol_wallet(ofield::borrow_mut<vector<u8>, Wallet>(
        &mut user.id,
        wallet_name,
    ), symbol);
}

public entry fun delete_wallet(user: &mut User, wallet_name: vector<u8>) {
    let Wallet { id,
        symbol: _,
        balance: _
    } = ofield::remove<vector<u8>, Wallet>(
        &mut user.id,
        wallet_name,
    );
    object::delete(id);
}

public entry fun reclaim_wallet(user: &mut User, wallet_name: vector<u8>, ctx: &mut TxContext) {
    let wallet = ofield::remove<vector<u8>, Wallet>(
        &mut user.id,
        wallet_name,

        transfer::transfer(wallet, tx_context::sender(ctx));
}
6. Generic Styles
6.1. In Rust
struct Gene<T> {
  name: T
}

impl<T> Gene<T> {
  pub fn new_gene(name: T) -> Gene<T> {
    Gene {
      name: name
    }
  }
}

fn main() {
  let gene = Gene::new_gene("gene");
  println!("gene name: {}", gene.name);
}
6.2. In Sui Move
module eamon::EAMON {
    struct Wallet<T: store + drop> has key, store {
        value: T
    }

    public fun create_wallet<T>(value: T): Wallet<T> {
        Wallet<T> { value }
    }

    let u64_wallet = Storage::create_wallet<u64>(1000000);
}
7. Witness
/// Module that defines a generic type `Guardian<T>` which can only be
/// instantiated with a witness.
module witness::EAMON {
    use sui::object::{Self, UID};
    use sui::transfer;
    use sui::tx_context::{Self, TxContext};

    /// Phantom parameter T can only be initialized in the `create_guardian`
    /// function. But the types passed here must have `drop`.
    struct Guardian<phantom T: drop> has key, store {
        id: UID
    }

    /// This type is the witness resource and is intended to be used only once.
    struct PEACE has drop {}

    /// The first argument of this function is an actual instance of the
    /// type T with `drop` ability. It is dropped as soon as received.
    public fun create_guardian<T: drop>(_witness: T, ctx: &mut TxContext): Guardian<T> {
        Guardian { id: object::new(ctx) }
    }

    /// Module initializer is the best way to ensure that the
    /// code is called only once. With `Witness` pattern it is
    /// often the best practice.
    fun init(witness: PEACE, ctx: &mut TxContext) {
        transfer::transfer(
            create_guardian(witness, ctx),
            tx_context::sender(ctx)
        )
    }
}
