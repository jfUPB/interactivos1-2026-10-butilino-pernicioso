# Unidad 3

## Bitácora de proceso de aprendizaje
### actividad 1
```py
def estado_nocturno(self,ev): #en este estado entramos, limpiamos los pixeles y medianten una condicion prendemos y apagamos con un timeout sin usar ciclos
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y+1,9)
            self.myTimer.start(self.timeInYellow)
        
        if ev == "Timeout":
            if display.get_pixel(self.x,self.y+1) == 0:
               display.set_pixel(self.x,self.y+1,9)
            else:
                display.set_pixel(self.x,self.y+1,0)
                
            self.myTimer.start(self.timeInYellow)
        
        if ev == "A":
            self.transicion_a(self.estado_waitInGreen)

```

### actividad 2

```py
if ev == "A" or ev == "B":
            self.secDes.append(ev)
            if len(self.secDes) == 3:
                if self.secDes == self.very:
                    self.transition_to(self.estado_config)
            else:
                self.secDes.clear()
```

notemos que la lista no se esta limpiando cuando pasamos de estado, por lo que cuando volvemos a este estado la lista ya esta llena, debemos agregar el el clear en el entry del estado

## Bitácora de aplicación 



## Bitácora de reflexión
