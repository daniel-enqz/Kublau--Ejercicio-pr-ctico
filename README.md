# Problema - LosMejoresLibros.com 📙

<img src="https://user-images.githubusercontent.com/72522628/230458972-37fd1766-871b-428b-8aff-6ed6c089eece.png" alt="kublau" width="300" height="100">

---

### Principales enfoques/features:

#### 📗 El cálculo de las regalías a los autores.
> Puntos importantes:
- El cálculo de las regalías debe tener en cuenta la tarifa fija que LosMejoresLibros.com cobra a los autores por cada libro vendido.
- La tarifa fija varía según el nivel de membresía actual del autor (gratuito, básico o premium).


#### 📗 La distribución de pagos de regalías a los autores.
> Puntos importantes:
- Pagar a los autores cada dos semanas por todas sus ventas hasta una semana antes de eso.
- Tener la flexibilidad de cambiar este horario; poder pagar a los autores una vez por semana en lugar de cada dos semanas.
---

# 📖 Solución 📖
- [Schema](#-schema)
- [Modelos y asociaciones](#-modelos-y-asociaciones)
- [Calculando pagos y distribuyendo regalías de manera eficiente](#-calculando-pagos-y-distribuyendo-regalías-de-manera-eficiente)
- [Posibilidad de cambiar los periodos de pago](#%EF%B8%8F-posibilidad-de-cambiar-los-periodos-de-pago)
- [Posibles Edge-Cases](#-posibles-edge-cases)
- [Nice To Haves](#-nice-to-haves-y-features-que-se-pdorían-explorar-con-más-detalle)

## 💾 Schema
- Como se menciona en el recuadro de la imagen, solo estamos considerando las tablas esenciales para resolver los problemas del cálculo y distribución de regalías.
- _Users_ y _Books_ son las tablas por defecto necesarias.
- _Payments:_ Nos será útil para tener registro de los pagos de regalías a los autores.
  - Campos muy importantes en esta tablas son: el día que se realizo la venta, el día que estamos pagando y la cantidad.
- _Sales:_ Por otro lado, esta tabla es importante para tener registro de las ventas que se han relizado, en esta tabla haremos el cálculo de la tarifra fija que tuvo cada venta. Tenerlo en una tabla es importante así nos aseguramos de cubrir el edge case de en caso de que haya cambios en las tarifas o que los precios de libros cambien durante las 2 semanas del periodo de ventas.
- _Purchases:_ Esta join table nos ayudará a llevar un registro de las compras que tengan los clientes y poder saber que libros tienen comprados en su cuenta.

<p align="center">
  <img src="https://user-images.githubusercontent.com/72522628/230512346-abe5b602-6bf5-4fd8-bd69-5606b43356ed.jpg" alt="schema" width="900" height="500">
</p>

---

## 🪣 Modelos y asociaciones
> Me gustaría detallar un poco de las asociaciones y como lucirían los modelos.

> Por el momento no detallaré demasiado las validaciones, scopes ó class/instance methdos; el própsito va más hacia como las tablas estarán ligadas. pues es importante que exista un flow correcto entre las instancias.

```ruby
class User < ApplicationRecord # Código ejemplo con configuración básica de devise
  devise :database_authenticatable, :registerable, :recoverable, :rememberable, :validatable, :confirmable, :lockable

  # Validaciones y relaciones comunes entre autores y clientes
  validates :first_name, presence: true
  validates :last_name, presence: true
end
```

```ruby
class Author < User
  has_many :books
  has_many :sales, through: :books
  has_many :payments

  validates :name, presence: true
  validates :membership_level, inclusion: { in: 0..2 }

  MEMBERSHIP_TYPES = { gratuito: 0, básico: 1, premium: 2 }.freeze
end
```

```ruby
class Client < User
  has_many :purchases
  has_many :sales, through: :purchases
  has_many :books, through: :sales
end
```

```ruby
class Book < ApplicationRecord
  belongs_to :author, class_name: "User"

  validates :title, presence: true
  validates :price, numericality: { greater_than: 0 }
end

```

```ruby
class Payment < ApplicationRecord
  belongs_to :author, class_name: "User"

  validates :payment_date, presence: true
  validates :amount, numericality: { greater_than: 0 }
end

```

```ruby
class Sale < ApplicationRecord
  belongs_to :book
  has_one :purchase
  has_one :client, through: :purchase, class_name: "User"
  has_one :author, through: :book, class_name: "User"

  validates :sale_date, presence: true
  validates :price, numericality: { greater_than: 0 }
  validates :fee, numericality: { greater_than_or_equal_to: 0 }

  before_create :calculate_fee

  private

  def calculate_fee
    membership_level = self.author.membership_level

    case membership_level
    when Author::MEMBERSHIP_TYPES[:gratuito]
      self.fee = price * 0.10
    when Author::MEMBERSHIP_TYPES[:básico]
      self.fee = price * 0.05
    when Author::MEMBERSHIP_TYPES[:premium]
      self.fee = price * 0.02
    else
      # [...]
    end
  end
end
```
---

## 💰 Calculando pagos y distribuyendo regalías de manera eficiente
> Para este proceso, ya que el pago de regalías será un proceso recurrente, utilizar sidekiq y sidekiq-scheduler con concurrencia es la opción. Sacando provecho de los workers, podríamos correr los cálculos de las regalías en paralelo. (whenever ó clockwork son otras opciones, pero considero que sidekiq schelduler es atinado por el momento.

```ruby
# Gemfile
gem 'sidekiq'
gem 'sidekiq-scheduler'

# config/sidekiq.yml
:concurrency: 25 # Importante para definir los "workers" y correr jobs en paralelo, más adelante detallo este uso.
:queues:
  - default

:schedule:
  payments_job:
    # Gracias a sidekiq-scheduler podemos jugar mucho la ejecución repetida de los jobs.
    cron: '0 0 * * 0,14' # Por ejemplo, este job se correra cada dos semanas. El día primeor de cada mes y los 15 de cada mes a media noche.
    class: PaymentsJob
```

> Con esta configuración podríamos crear un job con el nombre `jobs/payments_job.rb`.

#### 🍀Acerca de la ejucición en paralelo:
- Este job se encargaría de crear lotes de usarios en paralelo, dependiendo de la cantidad de autores en la aplicación, el número de lotes (batches), podría bajar o subir.
- Si la aplicación de 1 millón de autores, y colocamos el batch_size a 500, quiere decir que tendríamos 2,000 lotes en total (1,000,000 / 500).
- Con la concurrencia a 25, Sidekiq abrirá 25 threads en los que cada uno correra lotes de 500 autores al mismo tiempo.
- Aproximandamente estaremos procesando ~12,500 autores al mismo tiempo (25 * 500).
```ruby
class PaymentsJob < ApplicationJob
  queue_as :default

  def perform
    end_date = Date.today - 7.days
    # END_DATE: Caluclar la fecha final hasta una semana antes de la fecha de pago
    start_date = end_date - 13.days
    # START_DATE: Calcular el inicio del período de ventas de dos semanas de duración.

    batch_size = 500 # Definir el tamaño del lotes.
    Author.find_in_batches(batch_size: batch_size) do |authors_batch|
       # A este nuevo job le mandaríamos nuestros batches, para correr los pagos en paralelo
      BatchPaymenthJob.perform_async(authors_batch.map(&:id), start_date, end_date)
    end
  end
end
```

`jobs/batch_payments_job.rb`
> Las consultas a la base de datos es importante, por lo que es preferible hacer consultas directas con SQL

```ruby
class BacthPaymentJob < ApplicationJob
  queue_as :default

  def perform(author_ids, start_date, end_date)
    # Esta linea es muy importante, pues estamos sacando las ventas que tuvo el autor en el rango de fecha especificado y sumando (price - fee).
    # Recordemos que `fee` es fijo y varía dependiendo de la membresía del autor.
    authors = Author.where(id: author_ids)
                    .joins(books: :sales)
                    .select("authors.*, SUM(sales.price - sales.fee) as net_amount")
                    .where("sales.sale_date BETWEEN ? AND ?", start_date, end_date)
                    .group("authors.id")
                    .having("SUM(sales.price - sales.fee) > 0")

    authors.each do |author|
      total_payment = author.net_amount
      Payment.create!(author: author, payment_date: Date.today, amount: total_payment)
    end
  end
end
```

### 👀 IMPORTANTE: Los números de concurrencia y batch size podrían varirar y deberán ser monitoriados para no tener problemas con la conexión a la DB o demás problemas de procesamiento. El número de cores en nuestro servidor jugarán un gran papel, pero de momento dejemos estos números inciales.
---

## 🕰️ Posibilidad de cambiar los periodos de pago
Si bien anteriormente habíamos dejado una configuración default de pagos en la opción de `cron` de sidekiq y la variables, `end_date` y `start_date`
Tendríamos que hacer cambios en estos datos:
- Para esto, se podría utilizar las credentials de Rails y utilizar nuestra `master.key` para almacenar estas variables y cambiarlas facilemente.
```ruby
# config/credentials.yml.enc

payment_schedule:
  # 4: Jueves, 5: Viernes, 6: Sábado ....
  day_of_week: 5
  payment_interval: 1
```

> Al querer nosotros cambioar el periodo a por ejemplo 1 semana, tendríamos que hacer un cambio en la opción de cron: de sidekiq
```ruby
cron: '0 0 * * <%= Rails.application.credentials.dig(:payment_schedule, :day_of_week) %>'
```
> Así como cambiar las variables end_date y start_date.

```ruby
payment_interval = Rails.application.credentials.dig(:payment_schedule, :payment_interval)
end_date = Date.today - 7.days
start_date = end_date - (payment_interval.weeks - 1.day)
```
---

## 🦀 Posibles Edge-Cases
> Muchos de nuestros edge cases se presentan cuando se hacen cambios durante los periodos de venta, aquí agrego posibles soluciones a estos problemas:
> 🎨 Igualmente, coloco una de nuestras queries principales aquí, para  ser de ayuda en las siguienets explicaciones.
```ruby
authors = Author.where(id: author_ids)
                    .joins(books: :sales)
                    .select("authors.*, SUM(sales.price - sales.fee) as net_amount")
                    .where("sales.sale_date BETWEEN ? AND ?", start_date, end_date)
                    .group("authors.id")
                    .having("SUM(sales.price - sales.fee) > 0")
```

🤔 Es posible que los autores reembolsen alguna compra, podríamos agregar una columna refund en la tabla sales que indique si la venta ha sido reembolsada, al calcular las regalías, podemos excluir las ventas reembolsadas en la consulta. Esto cubriría el caso de que se haga un refund durante el periodo de dos semanas de ventas:
```ruby
.where("sales.sale_date BETWEEN ? AND ? AND sales.refund = ?", start_date, end_date, false)
```
En caso de que sea después, simplemente la venta se marca como refund y se hace la operación necesaria.
> Para atender el caso del refund, hace sentido hacer un service object que haga handle de esta acción.


🤔 Cambios en el período de pago: Pudimos lograr realización de esto, sin embargo puede ocurrir que si se cambia el periodo de pago durante las dos semanas de venta (o cualquiera que sea el rango), al momento de hacer la suma de las ventas, va a tomar las del rango actualizado y no todas las demas antes del cambio.
<br>
Para atender este caso, podríamos hacer que el cambio de estas variables en nuestro `config/credentials.yml.enc` se haga justo después de que se terminen los pagos. Es decir:<br>
1.- Los autores van a recibir el pago de sus ventas reailzadas en el periodo normal y en la fecha normal.<br>
2.- Justo cuando reciban su pago, se les enviará una notificación indicando el nuevo periodo de ventas y la nueva feacha de pago.<br>
3.- El nuevo periodo de ventas se actualizará cuando todos los autores hayan recibido su pago. Esto es una solución aproximada:

```ruby
[...]
authors.each do |author|
  total_payment = author.net_amount
  Payment.create!(author: author, payment_date: Date.today, amount: total_payment)
end
UpdatePayemntPlazo.permorm_now if payment_change.present?
```

🤔 Errores en optimización: Es posible que a pesar de la configuración de "lotes", paralelismo y concurrencia con `sidekiq`, `sidekiq-scheduler`, así como las consultas directas a la base de datos, pueda haber lentitud en el cálculo de regalías. Aquí se pueden explorar otras opciones como "redis" o técnicas de almacenamiento en caché o pre-agregación

---

## 🚀 Nice To Haves y Features que se pdorían explorar con más detalle
### (Por motivos de tiempo no pude detallar más de los siguientes temas que considero igual son importantes, así que colocaré una lista de estos temas)

- Administrate, ActiveAdmin, Rails-admin, blazer.
  - Proporcionar dashboards a los autores y a admins de la app.
- Tailwind
  - Posible framework a utilizar
- Memberships
  - Igualmente considero que me faltó detallar más de los beneficios que tendrían los autores con sus membresías como poder tener pagos por adelantado y demás beneficios pero por el momento se salen de nuestro objetivo.
- Payments
  - Un tema interesante también es la integración del sistema de pago, Stripe es siempre buena opción para recibir pagos de clientes.
- Sistema de Notificaciones y Mailers
- Similiar a como lo hace Medium.com, los autores pdrían tener sus propias membresías, así como los Clientes también.
- Considero que estas gemas, entre otras, podrían se incluídas en el desarrollo (better-errors, annotate, rollbar, bullet, paper-trail, money, strong-migrations)
