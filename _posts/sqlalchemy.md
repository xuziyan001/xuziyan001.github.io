#SQLAlchemy

###模型的映射方式

####1. Classical Mappings

一种经典的映射API，model与*TABLE*结构是分开的，他们之间通过*mapper()*函数关联。

	from sqlalchemy import Table, MetaData, Column, Integer, String, ForeignKey
	from sqlalchemy.orm import mapper

	metadata = MetaData()

	user = Table('user', metadata,
            Column('id', Integer, primary_key=True),
            Column('name', String(50)),
            Column('fullname', String(50)),
            Column('password', String(12))
        )

	class User(object):
    	def __init__(self, name, fullname, password):
        	self.name = name
        	self.fullname = fullname
        	self.password = password

	mapper(User, user)
	
通过*relationship()*可以将model和其他的表进行连接。

	address = Table('address', metadata,
            Column('id', Integer, primary_key=True),
            Column('user_id', Integer, ForeignKey('user.id')),
            Column('email_address', String(50))
            )

	mapper(User, user, properties={
    	'addresses' : relationship(Address, backref='user', 	order_by=address.c.id)
	})

	mapper(Address, address)

####2. Declarative Mapping
一种建立在Classical Mappings上的更加简洁和丰富的映射方式，model的声明和表结构同时被定义。

	from sqlalchemy.ext.declarative import declarative_base
	from sqlalchemy import Column, Integer, String, ForeignKey

	Base = declarative_base()

	class User(Base):
    	__tablename__ = 'user'

    	id = Column(Integer, primary_key=True)
    	name = Column(String)
    	fullname = Column(String)
    	password = Column(String)
    	
如果要定义额外的属性，映射到其他表结构上，可以在类定义中内联声明。

	class User(Base):
    	__tablename__ = 'user'

    	id = Column(Integer, primary_key=True)
    	name = Column(String)
    	fullname = Column(String)
    	password = Column(String)

    addresses = relationship("Address", backref="user", order_by="Address.id")

	class Address(Base):
    	__tablename__ = 'address'

    	id = Column(Integer, primary_key=True)
    	user_id = Column(ForeignKey('user.id'))
    	email_address = Column(String)
    	




