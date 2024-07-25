## 番外篇：我在dacade赚了100SUI

事先说明，本篇文章不是广告，只是讲述我如何在Dacade的挑战中赢得了100SUI的经历。如果大家有疑问可以自己去dacade官网查看，仔细辨别：https://dacade.org/communities/sui/challenges/19885730-fb83-477a-b95b-4ab265b61438

这是我在dacede中提现100SUI的截图:

![](https://github.com/Crazyjs123/crazyjs123.github.io/blob/main/pic/dacade.jpg?raw=true)

这个挑战的要求是

- 在DEFI领域创建创新项目，例如用于交易所，贷款平台或产量耕作应用程序的原型。

- 确保项目的高质量代码：结构良好，高效，最佳实践，详细的代码注释并没有错误。

按照上述要求编写，70分以上就能获得100SUI。他们的评分分为两个部分，一个是原创分满分60，另外一个部分则是代码质量分满分40分。原创是非常重要的分数，建议大家在写之前看看历次已经得分的提交，如果有人写过的功能最好就不要写了，原创分低的话很难超过70。高质量代码他们也给出了范例，可以拉优秀的代码下来研究一下再写

![](https://github.com/Crazyjs123/crazyjs123.github.io/blob/main/pic/best_submit.jpg?raw=true)代码提写完代码提交到代码仓库，并在dacade中提交你的代码仓库地址，接下来只需要等待审核就可以啦。根据我的观察审核周期是在一周到一个月不等，只要审核通过就会收到邮件。

下面我介绍一下我在dacade审核通过的代码，希望能给大家一起启发：

我编写的是一个公平公开的租房平台，可以防止提灯定损。在这平台中有三个角色：平台管理员，房东，房客。房东发布租房信息，房客租房时需要缴纳房租和押金，房租将会打款给房东，而押金则交由平台托管。等房租到期房东验房并管理员审核之后才能扣除押金，房客退房时退还剩余押金。

租房平台对象：

```rust
public struct RentalPlatform has key {
    // uid of the RentalPlatform object
    id: UID,
    // deposit stored on the rental platform, key is house object id,value is the amount of deposit
    deposit_pool: Table<ID, u64>,
    balance: Balance<SUI>,
    // rental notices on the platform, key is house object id
    notices: Table<ID, RentalNotice>,
    //owner of platform
    owner: address,
}
```

这是一个共享对象，不仅保存了房东发布的租房信息（notices）,还保存了租房押金，balance就是所有租房押金的总和，deposit_pool保存了每个房屋的押金明细。

还有一个平台管理员的对象，是一个独有对象，创建它主要是为了权限管理，只有拥有这个对象的上下文才能请求比如审核验房报告的接口。这个对象是在创建租房平台的接口中创建并转交给请求者的。

```rust
//presents Rental platform administrator
public struct Admin has key, store {
	// uid of admin object
	id: UID,
}
```

租房信息由房东发布，包含租金押金等信息：

```rust
//If the landlord wants to rent out a house, they first need to issue a rental notice
public struct RentalNotice has key,store  {
    // uid of the RentalNotice object
    id: UID,
    // the amount of gas to be paid per month
    monthly_rent: u64,
    // the amount of gas to be deposited 
    deposit: u64,
    // the id of the house object
    house_id: ID,
    // account address of landlord
    landlord: address,
}
```

房屋对象则是房屋的虚拟化，租房前房东拥有它，租房后租客拥有它：

```rust
// present a house object
public struct House has key {
    // uid of the house object
    id: UID,
    // The square of the house area
    area: u64,
    // The owner of the house
    owner: address,
    // A set of house photo links
    photo: String,
    // The landlord's description of the house
    description: String
}
```

租客想租房，就需要签订租房合同，租房合同一旦创建不可改变，不可销毁，所以这里的租房合同是一个不可变对象：

```rust
// present a house rentle contract object
public struct Lease has key,store {
    // uid of the Lease object
    id: UID,
    //uid of house object
    house_id: ID,
    // Tenant's account address
    tenant: address,
    // Landlord's account address
    landlord: address,
    // The month plan to rent
    tenancy: u32,
    // The mount of gas already paid
    paid_rent: u64,
    // The mount of gas already paid for deposit
    paid_deposit: u64,
}
```

在租约到期之后，房东需要出具验房报告并提交给管理员审核，以获取房屋损坏的赔偿。验房报告包含房屋损伤相关的证明

```rust
//presents inspection report object.The landlord submits the inspection report, and the administrator reviews the inspection report
public struct Inspection has key,store {
    // uid of the Inspection object
    id: UID,
    //id of the house object
    house_id: ID,
    //id of the lease object
    lease_id: ID,
    //Damage level, from 0 to 3, evaluated by the landlord
    damage: u8,
    //Description of damage details submitted by the landlord
    damage_description: String,
    //Photos of the damaged area submitted by the landlord
    damage_photo: String,
    //Damage level evaluated by administrator
    damage_assessment_ret: u8,
    //Deducting the deposit based on the damage to the house
    deduct_deposit: u64,
    //Used to mark whether the administrator reviews or not
    review_status: u8,
}
```

最后我再挑选几个主要的接口讲解一下。分别是房东发布租房信息、租客交钱签订租房合同、房东交房给租客、房东验房并提交报告、平台管理员审核验房报告并给与房东补偿，租客退房并领取剩余押金。

- 房东发布租房信息

  ```rust
  //The landlord releases a rental message, creates a rentalnotice object and create a  house object
  public fun post_rental_notice(platform: &mut RentalPlatform, monthly_rent: u64, housing_area: u64, description: vector<u8>, photo: vector<u8>, ctx: &mut TxContext): House {
      //caculate deposit by monthly_rent
      let deposit = (monthly_rent * DEPOSIT_PERCENT) / 100;
  
      let house = House {
          id: object::new(ctx),
          area: housing_area,
          owner: tx_context::sender(ctx),
          photo: string::utf8(photo),
          description:string::utf8(description),
      };
      let rentalnotice = RentalNotice{
          id: object::new(ctx),
          deposit: deposit,
          monthly_rent: monthly_rent,
          house_id: object::uid_to_inner(&house.id),
          landlord: tx_context::sender(ctx),
      };
  
      table::add<ID, RentalNotice>(&mut platform.notices, object::uid_to_inner(&house.id), rentalnotice);
  
      house
  }
  ```

- 租客交钱签订租房合同

  这个接口暂时只接收SUI币的coin,如果缴纳的金额不正确将会报错。在这个事务中，一旦租房成功，租房合约签订完毕，租房信息将会被删除。押金归入到了平台的balance字段，租金打入了房东账户。

  ```rust
  //call pay_rent function,transfer rent coin object  to landlord, deposit will be managed by platform.
  public entry fun pay_rent_and_transfer(platform: &mut RentalPlatform, house_address: address, tenancy: u32,  paid: Coin<SUI>, ctx: &mut TxContext) {
      let house_id: ID = object::id_from_address(house_address);
      let (rent_coin, deposit_coin, landlord) = pay_rent(platform, house_id, tenancy, paid, ctx);
      transfer::public_transfer(rent_coin, landlord);
      balance::join(&mut platform.balance, coin::into_balance(deposit_coin));
  }
  //Tenants pay rent and sign rental contracts
  public fun pay_rent(platform: &mut RentalPlatform, house_id: ID, tenancy: u32,  paid: Coin<SUI>, ctx: &mut TxContext): (Coin<SUI>, Coin<SUI>, address) {
      assert!(tenancy > 0, ETenancyIncorrect);
      assert!(table::contains<ID, RentalNotice>(&platform.notices, house_id), EInvalidNotice);
  
      let notice = table::borrow<ID, RentalNotice>(&platform.notices, house_id);
      assert!(!table::contains<ID, u64>(&platform.deposit_pool, notice.house_id), EInvalidHouse);
  
  
      let rent = notice.monthly_rent * (tenancy as u64);
      let total_fee = rent + notice.deposit;
      assert!(total_fee == coin::value(&paid), EInvalidSuiAmount);
  
      //the deposit is stored by rental platform
      let deposit_coin = coin::split<SUI>(&mut paid, notice.deposit, ctx);
      table::add<ID, u64>(&mut platform.deposit_pool, notice.house_id, notice.deposit);
  
      //lease is a Immutable object
      let lease = Lease {
          id: object::new(ctx),
          tenant: tx_context::sender(ctx),
          landlord: notice.landlord,
          tenancy: tenancy,
          paid_rent: rent,
          paid_deposit: notice.deposit,
          house_id: notice.house_id,
      };
      transfer::public_freeze_object(lease);
  
      //remove notice from platform
      let RentalNotice{id: notice_id, monthly_rent: _, deposit: _, house_id: _, landlord: landlord } = table::remove<ID, RentalNotice>(&mut platform.notices, house_id);
      object::delete(notice_id);
  
  
      (paid, deposit_coin, landlord)
  }
  ```

- 房东交房给租客

  这个接口非常简单，就是转移house的所有权给租客

  ```rust
  //After the tenant pays the rent, the landlord transfers the house to the tenant
  public entry fun transfer_house_to_tenant(lease: &Lease, house: House) {
      transfer::transfer(house, lease.tenant)
  }
  ```

- 房东验房并提交报告

  房东需要对房屋损伤评级，1-4级分别赔偿押金的0%,10%,50%,100%。具体赔付结果需要管理员定级。

  ```rust
  //Rent expires, landlord inspects and submits inspection report
  public entry fun landlord_inspect(lease: &Lease, damage: u8, damage_description: vector<u8>, damage_photo: vector<u8>, ctx: &mut TxContext) {
      assert!(lease.landlord == tx_context::sender(ctx), ENoPermission);
      assert!(damage >= DAMAGE_LEVEL_0 && damage <= DAMAGE_LEVEL_3, EDamageIncorrect);
      let inspection = Inspection{
          id: object::new(ctx),
          house_id: lease.house_id,
          lease_id: object::uid_to_inner(&lease.id),
          damage: damage,
          damage_description: string::utf8(damage_description),
          damage_photo: string::utf8(damage_photo),
          damage_assessment_ret: DAMAGE_LEVEL_UNKNOWN,
          deduct_deposit: 0,
          review_status: WAITING_FOR_REVIEW
      };
  
      transfer::public_share_object(inspection);
  }
  ```

- 平台管理员审核验房报告并给与房东补偿

  这个接口只有管理员可以请求，在这个事务里将审核验房报告，并对房屋评级、取出这个房屋的押金，根据评级结果赔偿一部分给房东、deposit_pool也要同步减去已经取走的押金。

  ```rust
      //The platform administrator reviews the inspection report and deducts the deposit as compensation for the landlord
  public entry fun review_inspection_report(platform: &mut RentalPlatform, lease: &Lease, inspection: &mut Inspection, _: &Admin, damage: u8, ctx: &mut TxContext)  {
      assert!(lease.house_id == inspection.house_id, EWrongParams);
      assert!(inspection.review_status == WAITING_FOR_REVIEW, EInspectionReviewed);
      assert!(table::contains<ID, u64>(&platform.deposit_pool, lease.house_id), EInvalidDeposit);
  
      let deduct_deposit:u64 = calculate_deduct_deposit(lease.paid_deposit, damage);
      let deposit_amount = table::borrow_mut<ID, u64>(&mut platform.deposit_pool, lease.house_id);
  
      assert!(deduct_deposit <= balance::value<SUI>(&platform.balance), EInsufficientBalance);
  
      inspection.damage_assessment_ret = damage;
      inspection.review_status = REVIEWED;
      inspection.deduct_deposit = deduct_deposit;
  
      if (deduct_deposit > 0) {
          *deposit_amount = *deposit_amount - deduct_deposit; 
  
          let deduct_coin = coin::take<SUI>(&mut platform.balance, deduct_deposit, ctx);
          transfer_deposit(deduct_coin, lease.landlord)
      };
  }
  
  ```

  

- 租客退房并领取剩余押金

  租客将房屋退还给房东，领取部分押金。清除租房平台押金和押金明细。

```rust
//The tenant returns the room to the landlord,collects deposit 
public entry fun tenant_return_house_and_transfer(platform: &mut RentalPlatform, lease: &Lease, house: House, ctx: &mut TxContext) {
    let house = tenant_return_house(platform, lease, house, ctx);

    transfer::transfer(house, lease.landlord)
}
//The tenant returns the room to the landlord and receives the deposit
public fun tenant_return_house(platform: &mut RentalPlatform, lease: &Lease, house: House, ctx: &mut TxContext): House {
    assert!(lease.house_id == object::uid_to_inner(&house.id), EWrongParams);
    assert!(lease.tenant == tx_context::sender(ctx), ENoPermission);
    assert!(table::contains<ID, u64>(&platform.deposit_pool, lease.house_id), EInvalidDeposit);

    let deposit = table::borrow(&platform.deposit_pool, lease.house_id);
    assert!(*deposit <= balance::value<SUI>(&platform.balance), EInsufficientBalance);

    //If there is still any remaining deposit, refund it to the tenant
    if (*deposit > 0) {
        let deposit_coin = coin::take<SUI>(&mut platform.balance, *deposit, ctx);
        transfer_deposit(deposit_coin, tx_context::sender(ctx));
    };

    let _ = table::remove<ID, u64>(&mut platform.deposit_pool, lease.house_id);

    house
}
```



其实这种赢奖励的机会并不少，大家在学习Web3的过程中应该积极参与这些活动，用金钱激励自己才能学的更快更深刻。

了解更多活动：

- telegram: t.me/move_cn
- QQ群: 79489587

