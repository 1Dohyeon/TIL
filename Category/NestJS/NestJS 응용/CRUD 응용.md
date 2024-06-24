## OneToOne에서의 CRUD

## OneToMany에서의 CRUD

## ManyToMany에서의 CRUD

#### Entity

ManyToMany란 테이블 A의 한 튜플이 테이블 B의 여러 튜플의 특정 컬럼을 참조할 수 있으며, 테이블 B의 한 튜플도 테이블 A의 여러 튜플의 특정 컬럼을 참조할 수 있는 경우이다. 예를 들어 사용자는 여러 주소를 자신의 프로필에 저장할 수 있고, 상세주소가 아닌 시/도 군/구의 주소라면 여러 사용자를 참조할 수 있다.

``` ts
@Entity({
  name: 'users',
})
export class UserEntity extends CommonEntity {
  @IsEmail({}, { message: '올바른 형태의 이메일을 작성해주세요.' })
  @IsNotEmpty({ message: '이메일을 작성해주세요.' })
  @Column({ type: 'varchar', unique: true, nullable: false })
  email: string;

  @IsString()
  @IsNotEmpty()
  @Column({ type: 'varchar', nullable: false })
  password: string;

  @IsString()
  @IsNotEmpty()
  @Column({ type: 'varchar', nullable: false })
  username: string;

  @IsString()
  @IsNotEmpty()
  @Column({ type: 'varchar', unique: true, nullable: false })
  nickname: string;
  
  @IsBoolean()
  @Column({ type: 'boolean', default: false })
  isAdmin: boolean;

  @ManyToMany(() => AddressEntity, (address) => address.users)
  @JoinTable({
    name: 'usersaddress',
    joinColumn: { name: 'userid', referencedColumnName: 'id' },
    inverseJoinColumn: { name: 'addressid', referencedColumnName: 'id' },
  })
  addresses: AddressEntity[];
}
```
코드를 통해서 설명하자면 아래 User Entity는 Address Entity를 참조하는 것을 알 수 있다. ManyToMany에서는 `@JoinTable`을 이용하여 두 테이블 간의 중간 테이블을 만들어서 데이터를 저장하는 것이 좋다. 테이블 구조는 아래 이미지와 같다.

![[Pasted image 20240624201700.png]]
address와 users 테이블 간 사이에 중간 테이블을 두어 각각의 PK를 참조하게 하였다. 아래는 Address Entity의 코드이다.

``` ts
@Entity('address')
export class AddressEntity extends CommonEntity {
  @PrimaryColumn({ type: 'varchar', nullable: false })
  city: string;

  @PrimaryColumn({ type: 'varchar', nullable: false })
  district: string;

  @ManyToMany(() => UserEntity, (user) => user.addresses)
  users: UserEntity[];
}
```
User Entity에서 중간 테이블을 만들어 두었기 때문에 이곳에서는 ManyToMany 관계만 지정하였다.

> 실무에서는 ManyToMany를 쓰지 않고, 직접 중간 테이블을 만들고 OneToMany, ManyToOne을 사용하는 것이 좋다.

#### CREATE

``` ts
async addAddress(userId: number, address: string): Promise<AddressEntity[]> {
    const user = await this.profileRepository.getProfileById(userId);

    if (!user) {
      throw new Error('사용자를 찾을 수 없습니다.');
    }

    if (user.addresses.length >= 3) {
      throw new Error('주소는 최대 3개까지만 설정할 수 있습니다.');
    }

	// 입력받은 string을 split함
    const { city, district } = this.parseAddress(address);
    
    const addressEntity = await this.addressRepository.findAddress(
      city,
      district,
    );

    if (!addressEntity) {
      throw new Error('존재하지 않는 주소입니다.');
    }

    this.checkAddress(user, addressEntity); // user가 이미 등록한 주소인지 체크
    return await this.profileRepository.addAddress(user, addressEntity);
  }
```
위 코드는 사용자가 자신의 프로필에 주소를 등록할 때 사용되는 service 코드이다. 코드 파라미터를 보면 알겠지만, 사용자는 string 타입으로 주소 정보를 넘기는 형태라 이를 처리하고, 서비스 룰에 맞도록 몇몇 예외처리를 해두었다. 직접적으로 DB에 주소 정보를 추가하는 로직은 repository 코드 파일에서 만들었다.

``` ts
async addAddress(
    user: UserProfile,
    address: AddressEntity,
  ): Promise<AddressEntity[]> {
    try {
      await this.repository
        .createQueryBuilder()
        .relation(UserEntity, 'addresses')
        .of(user.id)
        .add(address.id);

      // 사용자 주소 목록 갱신
      const updatedUser = await this.getProfileById(user.id);
      return updatedUser.addresses;
    } catch (err) {
      throw new Error(`주소 추가 중 에러 발생: ${err.message}`);
    }
  }
```
쿼리문을 통해서 사용자 객체 user의 addresses에 직접 데이터를 넣는다. `.of`를 통해서 사용자를 지정하고 `.add`를 통해서 삽입할 값을 넣는다. `user.id`, `address.id`를 포함하는 usersaddress 테이블에 값을 집어넣는 것이다. 이렇게 되면 두 테이블의 레코드는 서로 연결될 수 있다.

위 쿼리 문 말고 아래 쿼리문을 사용해도 된다.

``` ts
      await this.repository
        .createQueryBuilder()
        .insert()
        .into('usersaddress')
        .values({ userid: user.id, addressid: address.id })
        .execute();
```
중간 테이블의 이름을 알고 있다면 위 쿼리문이 좀 더 직관적으로 보인다.


#### GET

바로 repository 코드를 보여주자면 아래와 같다.

``` ts
  async getAddressByUserId(
    userId: number,
  ): Promise<AddressEntity[] | undefined> {
    const user = await this.getProfileById(userId);
    return user.addresses;
  }
```
사용자는 addresses라는 배열을 가져 자신의 주소 정보에 접근할 수 있기에 쿼리에서 SELECT를 하지않고 그냥 user.addresses만 반환하면 된다.

#### DELETE

``` ts
async removeAddress(
    user: UserProfile,
    addressId: number,
  ): Promise<AddressEntity> {

    // addressId가 숫자인지 확인
    const addressIdToNumber = Number(addressId);
    console.log(addressIdToNumber);
    console.log(user.addresses);

    const addressToRemove = user.addresses.find(
      (address) => address.id === addressIdToNumber,
    );

    if (!addressToRemove) {
      throw new Error('주소가 없습니다.');
    }

    try {
      await this.repository
        .createQueryBuilder()
        .relation(UserEntity, 'addresses')
        .of(user.id)
        .remove(addressId);

      return addressToRemove;
    } catch (err) {
      throw new Error(`저장 중 에러 발생: ${err.message}`);
    }
  }
```
데이터를 삽입할 때와 똑같이 쿼리문(`.remove`)을 사용해서 특정 데이터를 삭제하였다. 원래는 filter를 사용해서 특정 id를 필터링하고 addresses 갱신하는 방법을 택했었는데, 이렇게 되면 usersaddresses에서 중복 에러가 발생하였다. 찾아보니 삭제한 데이터를 제외하고 기존 주소 정보를 다시 INSERT를 하여 중복 에러가 발생했던 것이다. 따라서 쿼리문을 통해서 직접적으로 제거하였다.

#### UPDATE

``` ts
async updateAddress(
    user: UserProfile,
    oldAddressId: number,
    newAddressEntity: AddressEntity,
  ): Promise<AddressEntity[]> {
    try {
      await this.removeAddress(user, oldAddressId);
      await this.addAddress(user, newAddressEntity);

      const updatedUser = await this.getProfileById(user.id);
      return updatedUser.addresses;
    } catch (err) {
      throw new Error(`주소 업데이트 중 에러 발생: ${err.message}`);
    }
  }
```
업데이트는 위 CREATE와 DELETE를 이용하여 삭제 후에 새롭게 생성하는 것으로 바꾸었다. 이것도 user.addresses에 접근하여 특정 address의 id부분만 바꾸려고 하였더니 중간 테이블에서 중복 에러가 발생하였다. 따라서 쿼리문으로 update를 실행해봤는데, 여기서도 중복 에러가 발생하여 삭제 후 새로운 데이터를 생성하는 것으로 남겨두었다.

userid는 어차피 같고 addressid만 변경하면 되기 때문에 비효율적인 것 같아서 update에서 나오는 에러를 빨리 해결해 봐야할 것 같다.
