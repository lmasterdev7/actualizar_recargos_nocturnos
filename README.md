# ⭐ Actualizar Recargos Nocturnos para el año 2026 ⭐
**El objetivo de este script es actualizar los recargos automáticamente y en segundos desde la base de datos, se configuró para abarcar al mayor número de empresas posible, y las configuraciones más comunes del Nomiplus para los clientes de Ingecon Group SAS**

Antes de ejecutar el script siga las intrucciones preliminares:
## 1️⃣ Realice una copia de seguridad de la base de datos
## 2️⃣ Consulte los niveles de tiempo,
Abra SQL Managment Studio luego y ejecute la siguiente consulta para obtener los KeyLevel de los niveles de tiempo:
```sql
SELECT * FROM catTimeLevels
```
Modifique la primera parte del script donde se están declarando los niveles de tiempo y cambie el "?" por el KeyLevel que corresponda
## 3️⃣ Con relación al Dominical Diurno y Dominical Nocturno,
Si no hay niveles de tiempo diferentes para los dominicales, cambiar el ? por NULL en @dd y @dn. En caso que exista el @dd pero no @dn, deberá indicar el nivel de tiempo que corresponda a @dd, pero el @dn no puede dejarlo en NULL, lo puede reemplazar por el mismo KeyLevel de @fn  
## 4️⃣ Ejecute el script y luego revise dos o tres políticas para comprobar que los cambios se aplicaron correctamente
---
~~~sql
USE TASTD

-- Declara las variables, debe ingresar el id del nivel de tiempo antes de ejecutar el script
DECLARE 
  @hdo  INT = ?,
  @rn   INT = ?,
  @hed  INT = ?,
  @hen  INT = ?,
  @fd   INT = ?,
  @fn   INT = ?,
  @hefd INT = ?,
  @hefn INT = ?,
  @dd   INT = ?,
  @dn   INT = ?,

  @KeyPolicy INT,
  @Shift INT,
  @ShiftDescription VARCHAR(100),
  @Start INT,
  @Start_Keylevel INT,
  @Finish INT,
  @Finish_Keylevel INT,
  @Limit INT,
  @Lunch INT;

/*********************************************************************************************************/
-- 1. Actualiza todos los puntos amarillos que se encuentran en las 21:00 hacia 19:00 Si son RN, FN y DN
BEGIN
UPDATE p
  SET p.ExpectedTime=1140
    FROM detShiftsPairs p
      LEFT JOIN detShifts s ON p.KeyPolicy=s.KeyPolicy AND p.Shift = s.Shift
        WHERE p.ExpectedTime=1260 AND p.KeyLevel IN (@rn, @fn, @dn) 
          AND p.IsSystem=0  AND s.Start NOT IN (1140) AND s.Finish NOT IN (1140);
END


/*********************************************************************************************************/
-- 2. Actualiza HDO->RN, HED->HEN, FD->FN, DD->DN y HEFD->HEFN en marcaciones de entrada que van de 19:00 - 20:59
BEGIN
UPDATE detShiftsPairs 
  SET KeyLevel=@rn
    WHERE IsSystem=1 AND ExpectedTime BETWEEN 1140 AND 1259 AND keylevel= @hdo;
END

BEGIN
UPDATE detShiftsPairs 
  SET KeyLevel=@hen
    WHERE IsSystem=1 AND ExpectedTime BETWEEN 1140 AND 1259 AND keylevel= @hed;
END


BEGIN
UPDATE detShiftsPairs 
  SET KeyLevel=@fn
    WHERE IsSystem=1 AND ExpectedTime BETWEEN 1140 AND 1259 AND keylevel=@fd;
END

BEGIN
UPDATE detShiftsPairs 
  SET KeyLevel=@dn
    WHERE IsSystem=1 AND ExpectedTime BETWEEN 1140 AND 1259 AND keylevel=@dd;
END

BEGIN
UPDATE detShiftsPairs 
  SET KeyLevel=@hefn
    WHERE IsSystem=1 AND ExpectedTime BETWEEN 1140 AND 1259 AND keylevel= @hefd;
END



/*********************************************************************************************************/
-- 3. Actualiza HDO->RN, HED->HEN, FD->FN, DD->DN y HEFD->HEFN en marcaciones de salida que van de 19:00 - 20:59
BEGIN
UPDATE detShiftsPairs 
  SET KeyLevel=@rn
    WHERE IsSystem=2 AND ExpectedTime BETWEEN 1140 AND 1259 AND keylevel=@hdo;
END

BEGIN
UPDATE detShiftsPairs 
  SET KeyLevel=@hen
    WHERE IsSystem=2 AND ExpectedTime BETWEEN 1140 AND 1259 AND keylevel=@hed;
END

BEGIN
  UPDATE detShiftsPairs 
    SET KeyLevel=@fn
      WHERE IsSystem=2 AND ExpectedTime BETWEEN 1140 AND 1259 AND keylevel=@fd;
END

BEGIN
  UPDATE detShiftsPairs 
    SET KeyLevel=@dn
      WHERE IsSystem=2 AND ExpectedTime BETWEEN 1140 AND 1259 AND keylevel=@dd;
END

BEGIN
  UPDATE detShiftsPairs 
    SET KeyLevel=@hefn
      WHERE IsSystem=2 AND ExpectedTime BETWEEN 1140 AND 1259 AND keylevel=@hefd;
END


/*********************************************************************************************************/
-- 4. Actualiza HDO->RN, HED->HEN, FD->FN, DD->DN y HEFD->HEFN en puntos amarillos que van de 19:00 - 20:59
BEGIN
  UPDATE detShiftsPairs 
    SET KeyLevel=@rn
      WHERE IsSystem=0 AND ExpectedTime BETWEEN 1140 AND 1259 AND keylevel=@hdo;
END

BEGIN
  UPDATE detShiftsPairs 
    SET KeyLevel=@hen
      WHERE IsSystem=0 AND ExpectedTime BETWEEN 1140 AND 1259 AND keylevel=@hed;
END

BEGIN
  UPDATE detShiftsPairs 
    SET KeyLevel=@fn
      WHERE IsSystem=0 AND ExpectedTime BETWEEN 1140 AND 1259 AND keylevel=@fd;
END

BEGIN
  UPDATE detShiftsPairs 
    SET KeyLevel=@dn
      WHERE IsSystem=0 AND ExpectedTime BETWEEN 1140 AND 1259 AND keylevel=@dd;
END

BEGIN
  UPDATE detShiftsPairs 
    SET KeyLevel=@hefn
      WHERE IsSystem=0 AND ExpectedTime BETWEEN 1140 AND 1259 AND keylevel=@hefd;
END
/*********************************************************************************************************/
-- Cuando no exista una marcación a las 19:00, se agregará una con el nivel de tiempo que corresponda
BEGIN
DECLARE cursor_origen CURSOR LOCAL FAST_FORWARD FOR
	SELECT DISTINCT  s.KeyPolicy, s.Shift, s.ShiftDescription, s.Start, st.start_keylevel, s.Finish, fn.finish_keylevel, IIF(l.Limit>0, l.Limit, 0) AS Limit, IIF(c.Lunch>0, c.Lunch, 0) AS Lunch
		
	FROM detShifts s
		LEFT JOIN detShiftsPairs p ON s.KeyPolicy=p.KeyPolicy AND s.Shift=p.Shift
		LEFT JOIN detShiftsDeductions d ON s.KeyPolicy=d.KeyPolicy AND s.Shift=d.Shift
		LEFT JOIN (SELECT DISTINCT KeyPolicy, Shift, SUM(Deduction) AS Lunch
						FROM detShiftsDeductions
							GROUP BY KeyPolicy, Shift) c ON s.KeyPolicy=c.KeyPolicy AND s.Shift=c.Shift
		LEFT JOIN (SELECT l.KeyLimitDef ,l.Description, l1.Limit
						FROM catLimitDef l
							LEFT JOIN detLimitDef l1 ON l.KeyLimitDef=l1.keyLimitDef
								WHERE l1.IsTimeForDay=1) l ON s.keyTLLimDef=l.KeyLimitDef
		LEFT JOIN (SELECT p1.keylevel AS start_keylevel, s1.KeyPolicy, s1.Shift
						FROM detShifts s1
							LEFT JOIN detShiftsPairs p1 ON s1.KeyPolicy=p1.KeyPolicy AND s1.Shift=p1.Shift AND s1.Start=p1.ExpectedTime
					) st ON s.KeyPolicy=st.KeyPolicy AND s.Shift=st.Shift
		LEFT JOIN (SELECT p2.keylevel AS finish_keylevel, s2.KeyPolicy, s2.Shift
						FROM detShifts s2
							LEFT JOIN detShiftsPairs p2 ON s2.KeyPolicy=p2.KeyPolicy AND s2.Shift=p2.Shift AND s2.Finish=p2.ExpectedTime
					) fn ON s.KeyPolicy=fn.KeyPolicy AND s.Shift=fn.Shift

			
			WHERE
				s.Start < 1140 AND
				NOT EXISTS (
					SELECT 1
						FROM detShiftsPairs x
							WHERE x.KeyPolicy=s.KeyPolicy
							AND x.Shift= s.Shift
							AND x.ExpectedTime=1140
				);

OPEN cursor_origen;
FETCH NEXT FROM cursor_origen INTO @KeyPolicy, @Shift, @ShiftDescription, @Start, @Start_Keylevel, @Finish, @Finish_Keylevel, @Limit, @Lunch;

WHILE @@FETCH_STATUS = 0
BEGIN
	-- Insertar HEN
	IF @Start_Keylevel IN (@hed, @rn) OR (@Finish_Keylevel=@hed AND @Finish<1140) OR (@Start_Keylevel=@hdo AND (@Start+@limit+@lunch < 1140))
	BEGIN
		INSERT INTO detShiftsPairs VALUES (@KeyPolicy, @Shift, 1140, 1, @hen, 0, 15, 15, 15 , 15, 30, 30, 0, 1)
	END
	-- Insertar HEFN
	ELSE IF @Start_Keylevel IN (@hefd, @fn) OR (@Finish_Keylevel=@hefd AND @Finish<1140) OR (@Start_Keylevel=@fd AND (@Start+@limit+@lunch < 1140))
	BEGIN
		INSERT INTO detShiftsPairs VALUES (@KeyPolicy, @Shift, 1140, 1, @hefn, 0, 15, 15, 15 , 15, 30, 30, 0, 1)
	END
	-- Insertar FN
	ELSE IF @Start_Keylevel IN (@fn, @fd) AND @Start+@limit+@lunch > 1140 
	BEGIN
		INSERT INTO detShiftsPairs VALUES (@KeyPolicy, @Shift, 1140, 1, @fn, 0, 15, 15, 15 , 15, 30, 30, 0, 1)
	END
	-- Insertar DN
	ELSE IF @Start_Keylevel IN (@dn, @dd) AND @Start+@limit+@lunch > 1140 
	BEGIN
		INSERT INTO detShiftsPairs VALUES (@KeyPolicy, @Shift, 1140, 1, @dn, 0, 15, 15, 15 , 15, 30, 30, 0, 1)
	END
	-- Insertar RN
	ELSE
	BEGIN
	INSERT INTO detShiftsPairs VALUES (@KeyPolicy, @Shift, 1140, 1, @rn, 0, 15, 15, 15 , 15, 30, 30, 0, 1)
	END

	FETCH NEXT FROM cursor_origen
	INTO @KeyPolicy, @Shift, @ShiftDescription, @Start, @Start_Keylevel, @Finish, @Finish_Keylevel, @Limit, @Lunch;

	END

	CLOSE cursor_origen
	DEALLOCATE cursor_origen

END
~~~
