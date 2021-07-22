# FlatBuffers

`FlatBuffers`是一个开源高效的序列化工具库，与`ProtocalBuffers`类似，同样需要`IDL`文件，提供了`C++, C#, C, Go, Java, PHP, Python，Javascript`的接口。

最初是为`Android`游戏和注重性能的应用而开发，最大特点是**反序列化**速度极快，或者可以说无需解码。

## FlatBuffers的优势

* **无需反序列化即可访问序列化数据** - FlatBuffers 的不同之处在于它在平面二进制缓冲区中表示分层数据，这样它仍然可以直接访问而无需解析/解包，同时仍然支持数据结构演化（向前/向后兼容）。
* **优秀的内存效率和速度**

## 编译器下载安装

- [flatc](https://github.com/google/flatbuffers/releases/tag/v2.0.0)

## IDL示例

```
// Example IDL file for our monster's schema.
 
namespace MyGame.Sample;
 
enum Color:byte { Red = 0, Green, Blue = 2 }
 
union Equipment { Weapon } // Optionally add more tables.
 
struct Vec3 {
  x:float;
  y:float;
  z:float;
}
 
table Monster {
  pos:Vec3; // Struct.
  mana:short = 150;
  hp:short = 100;
  name:string;
  friendly:bool = false (deprecated);
  inventory:[ubyte];  // Vector of scalars.
  color:Color = Blue; // Enum.
  weapons:[Weapon];   // Vector of tables.
  equipped:Equipment; // Union.
  path:[Vec3];        // Vector of structs.
}
 
table Weapon {
  name:string;
  damage:short;
}
 
root_type Monster;
```

对应的json：

```json
{
  "pos": {
    "x": 1.0,
    "y": 2.0,
    "z": 3.0
  },
  "hp": 300,
  "name": "Orc",
  "weapons": [
    {
      "name": "axe",
      "damage": 100
    },
    {
      "name": "bow",
      "damage": 90
    }
  ],
  "equipped_type": "Weapon",
  "equipped": {
    "name": "bow",
    "damage": 90
  }
}
```

## 序列化

```golang
package flatbuf

import (
	sample "GoEchoton/pkg/flatbuf/MyGame/Sample"

	flatbuffers "github.com/google/flatbuffers/go"
)

func NewDemo() {
	builder := flatbuffers.NewBuilder(1024)
	weaponOne := builder.CreateString("Sword")
	weaponTwo := builder.CreateString("Axe")

	sample.WeaponStart(builder)
	sample.WeaponAddName(builder, weaponOne)
	sample.WeaponAddDamage(builder, 3)
	sword := sample.WeaponEnd(builder)

	sample.WeaponStart(builder)
	sample.WeaponAddName(builder, weaponTwo)
	sample.WeaponAddDamage(builder, 5)
	axe := sample.WeaponEnd(builder)

	sample.MonsterStartWeaponsVector(builder, 2)
	builder.PrependUOffsetT(axe)
	builder.PrependUOffsetT(sword)
	weapons := builder.EndVector(2)

	name := builder.CreateString("Orc")

	sample.MonsterStartInventoryVector(builder, 10)
	for i := 9; i >= 0; i-- {
		builder.PrependByte(byte(i))
	}
	inv := builder.EndVector(10)

	sample.MonsterStartPathVector(builder, 2)
	sample.CreateVec3(builder, 1.0, 2.0, 3.0)
	sample.CreateVec3(builder, 4.0, 5.0, 6.0)
	path := builder.EndVector(2)

	sample.MonsterStart(builder)
	sample.MonsterAddPos(builder, sample.CreateVec3(builder, 1.0, 2.0, 3.0))
	sample.MonsterAddHp(builder, 300)
	sample.MonsterAddName(builder, name)
	sample.MonsterAddInventory(builder, inv)
	sample.MonsterAddColor(builder, sample.ColorRed)
	sample.MonsterAddWeapons(builder, weapons)
	sample.MonsterAddEquippedType(builder, sample.EquipmentWeapon)
	sample.MonsterAddEquipped(builder, axe)
	sample.MonsterAddPath(builder, path)
	orc := sample.MonsterEnd(builder)

	builder.Finish(orc)

	buf := builder.FinishedBytes()
}
```

* 需要使用一个`Builder`的生成区，并指定缓存区大小
* 写入缓存区以`XXXXStart(builder)`开启缓存区写入，以`XXXXEND(builder)`结束缓存区写入
* 以`Finish()`结束整个写入，这时候，调用`Start`就不行了
* `FinishedBytes()`最终输出结果


## 反序列化

```golang
// 直接从Root Type开始反序列化，如果序列化局部，如果要跳过加入的头，则指定第二个参数为非0
monster := sample.GetRootAsMonster(buf, 0)

// 直接访问元素
hp := monster.Hp()
mana := monster.Mana()
name := string(monster.Name())  // byte[]

// 子对象
pos := monster.Pos(nil)  // 非nil, 可以用已经生成的对象指针，空间重用
x := pos.X()
y := pos.Y()
z := pos.Z()

// Vector 访问
invLength := monster.InventoryLength()  // 长度
thirdItem := monster.Inventory(2)  // 索引访问

// 子对象 Vector
weaponLength := monster.WeaponsLength()

if monster.Weapons(weapon, 1) {
        secondWeaponName := weapon.Name()
        secondWeaponDamage := weapon.Damage()
}

// union 访问
// `monster.Equipped()` function.
unionTable := new(flatbuffers.Table)
// 赋值到指针
if monster.Equipped(unionTable) {
        unionType := monster.EquippedType()
 
        if unionType == sample.EquipmentWeapon {
                unionWeapon = new(sample.Weapon)
                unionWeapon.Init(unionTable.Bytes, unionTable.Pos)
 
                weaponName = unionWeapon.Name()
                weaponDamage = unionWeapon.Damage()
        }
}
```
