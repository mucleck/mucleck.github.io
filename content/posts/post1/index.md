---
title: "Renderizando 8 millones de pixeles en 130ms"
date: 2026-09-08
tags: ["Go"]
categories: ["Programming"]
---

![mandelbrot](thumbnail.png)
TL;DR Decide como paralelizar el trabajo, paraleliza, utiliza algun truco con la compresion de la imagen y usa blanco y negro en vez de RGBA
## ¿Que es el set de Mandelbrot?
Este es un fractal bastante conocido pero por si no lo conoces te dejo algunos links donde se explica muy bien que es y algunas curiosidades:
- [Numberphile](https://www.youtube.com/watch?v=FFftmWSzgmk&pp=ygUabWFuZGVsYnJvdCBzZXQgbWF0aCB2aWRlb3M%3D)
- [swap2](https://youtu.be/Ed1gsyxxwM0?si=FscwFbeNGuY4C63k)
- [Numberphile + Dr Holly Krieger](https://www.youtube.com/watch?v=4LQvjSf6SSw&t=541s&pp=ygUabWFuZGVsYnJvdCBzZXQgbWF0aCB2aWRlb3M%3D)

En pocas palabrasla imagen que ves es la representacion de aplicar de manera iterativa la funcion f(z) = z*z + c a todos los pixeles de un plano en el que representamos los numeros "normales" en el eje X y los imaginarios en el eje Y. El color que cada pixel tiene se decide por cuantas iteraciones tarda f(z) en superar un valor numerico que decidamos (-I en mi codigo por ejemplo1) ,o si tenemos suerte, el valor super rapidamente el valor 4. Si no se ha entendido, mira los videos que se explican mucho mejor que yo.

## Implemmentacion simple
El flujo para crear el la imagen es bastante simple (mas allá de cosas del lenguaje especificas, yo uso Golang por gusto y eficiencia) solo tenemos que recorrer todos los pixeles de la imagen y para cada uno escoger un valor de color RGBA. Por lo que una Implemmentacionrapida y simple del programa se vería así:
```Go
// ... crear la imagen 
var width, height int
for i := 0; i < width; i++ {
    for j := 0; j < height; j++ {
        color = getPixelColor(i,j)
    }
}
// ... encode 
```
Y realmente esto sería todo, luego en pixel color solo habría que mapear la coordenada con un punto en el plano complejo y nuestro programa funcionaría al momento.

¿Lo malo? Que esto es muy lento, haciendo la prueba en mi ordenador esto tarda alrededor de 2 segundos para una imagen en 4k. Vamos a ver como optimizar esto.

## Dividiendo la imagen en chunks
Claramente esta primera version es muy lenta ya que vamos iterando sobre cada pixel y el siguiente tiene que esperar a que uno haya acabado. Esto es más costoso para nuestro programa ya que or cada pixel la funcion getPixelColor hace algo así:
```Go
func getPixelColor(i, j int) uint8 {
    var c, z complex128
    c = mapCoordsToImaginaryNumber(i, j)

    // Iteramos sobre f(z) hasta salir del radio o superar el numero de iteraciones
    // Guardamos i para el smooth coloring 
    var i int

	for i = 0; i < config.Iterations && cmpl.Abs(z) <= 2; i++ {
		z = z*z + c
	}

    // según el valor de i calcularemos el color del pixel
 
}
```
Como se ve es mucho trabajo e iriamos mas rapido si pudieramos paralelizar esto. El primer paso para parelizar es dividir la imagen en regiones que luego podremos ir pasando para que diferentes hilos las vayan renderizando. Lo que necesitaremos en estas regiones será saber desde que x e y tenemos que empezar y hasta que x e y tenemos que llegar. Mi implementacion basica es la siguiente: 
```Go
type Region struct {
    x, y, maxX, maxY int
}

func generateRegions() []Region {
	var regions []Region
	for i := range Threads {
		x := i * (Width / Threads)
		maxX := (i + 1) * (Width / Threads)
		for j := range Threads {
			y, maxY := j*(Height/Threads), (j+1)*(Height/Threads)
			regions = append(regions, Region{x: x, maxX: maxX, y: y, maxY: maxY})
		}
	}

	return regions
}
```
Como veis nos repartimos toda la imagen en el numero de Threads que tengamos (en mi caso 8 pero se puede cambiar si se quiere) y ahora ya tenemos un array (se llaman slices en Go pero por ser mas generalista) con informacion en cada elemento de donde tenemos que empezary acabar para renderizar los pixeles. 
## Paralelizando 
En go es muy simple paralelizar una tarea(es fácil hacerlo pero también cuesta no cagarla con locks y demás problemas). Para hacerlo necesitamos un chanel que es como una pipe donde lanzaremos trabajos y las go routines(threads) cogeran uno de estos e iran ejecutando lo necesario. 

En codigo esto queda algo como lo siguiente:
```Go
//Nuestra pipe por donde nos irán entrando Regiones para renderizar
jobs := make(chan Region)

// ... logica para lanzar go routines por cada thread

for _, region := range regions {
    jobs <- region // Mandamos una Region 
}
```
Ahora tenemos nuestra pipe de donde podemos coger una Region e ir renderizandola en paralelo. Para implementar esto usaremos un WaitGroup e iremos lanzando por cada Thread una funcion paralela que cogera una region y esperara a que las demas acaben. En codigo:
```Go
var wg sync.WaitGroup
for range Threads { //Iteramos or n threads
    wg.Go(func(){ //Esta es la forma que tiene Go de lanzar go routines (para un wait group)
        for region := range jobs { //Recibimos una region a renderizar
            for i := region.x; i < region.maxX; i++{
                for j := region.y; j < region.maxY;  j++{
                    c := getPixelColor(i, j) //Lo mismo que antes
                    assignColorToPixel(i, j, c) //Pseudo codigo, asignamos un color al pixel
                }
            }
        }
    })

}
close(jobs)
wg.Wait() // Esperamos a que todos los hilos hayan acabado
```
Con esto ya obtenemos unos tiempos bastante rapidos, en mi caso al rededor de 400ms. A partir de aqui las dos optimizaciones que quedan son un poco controversiales ya que ahora vamos a afectar al resultado de la imagen.
 ## Quien ha pedido compresion
 400ms esta bastante bien para 8 millones de pixeles pero para mi caso de uso de despues que iba a ser hacer esto interactivo iba a ser un poco lento. Mirando el pprofile del programa (es la herramienta de go para ver donde se va el tiempo en tu programa) vi que 200ms se me estaban yendo la parte de comprimir la imagen final. Preguntandole a ChatGPT me comentó que si queria la imagen podía no tener compresion eso sí a cambio de pasar de una imagen de 580Kb a 29 Mb... Es un poco loco pero todo sea por la velocidad.

 Así que eso fue lo que hice aquí: 
```Go
	var compressionLevel png.CompressionLevel

	if config.pngCompression {
		compressionLevel = png.BestSpeed
	} else {
		compressionLevel = png.NoCompression
	}

	encoder := png.Encoder{
		CompressionLevel: compressionLevel,
	}

	err := encoder.Encode(file, image)
	if err != nil {
		return err
	}
	return nil

```
Como veis si la flag viene en false decido no comprimir la imagen y con esto bajamos a unos 200ms.
## Una ultima optimizacion
A la hora de asignar el color a un pixel tenemos que asignar 4 valores uno para cada letra del RGBA. Esto esta bien pero si por ejemplo como es mi caso solo soportas una escala de grises entonces es innecesario tener que hacer estas 4 asignaciones. De esta no pongo el codigo porque tampoco la he implementado ya que en un futuro quiero poder soportar diferentes escalas de colores pero si se aplica ahí estan los 130ms de media para una imagen en 4k, es decir 8,294 millones de pixeles.

El codigo está en aquí: [codigo](https://github.com/mucleck/fractals-generator) y cualquier sugerencia de mejora es más que bienvenida!
Dejo una imagen generada con este programa de una arte del set, es un fractal muy chulo.
![show](./show1.png)
<script type="text/javascript" src="https://www.freevisitorcounters.com/auth.php?id=b0008dabf54fa81daa92ba76c8b262e91ea5bb3d"></script>
<script type="text/javascript" src="https://www.freevisitorcounters.com/en/home/counter/1640154/t/3"></script>

