title
using System;

class Program
{
    static void Main()
    {
        Random rnd = new Random();

        // 1) Genero la cartella con 15 numeri distinti
        int[] cartella = GeneraCartella(rnd);

        // 2) Ordino la cartella (necessario per Binary Search)
        Array.Sort(cartella);

        Console.WriteLine("La tua cartella:");
        Console.WriteLine(string.Join(", ", cartella));

        bool[] marcati = new bool[15];  // tiene traccia dei numeri trovati
        int numeriTrovati = 0;

        Console.WriteLine("\nInizio estrazioni...\n");

        // 3) Estrazione numeri finché non fai tombola
        while (numeriTrovati < 15)
        {
            int estratto = rnd.Next(1, 91);
            Console.WriteLine($"Estratto: {estratto}");

            // 4) Verifico con BINARY SEARCH
            int index = Array.BinarySearch(cartella, estratto);

            if (index >= 0 && !marcati[index])
            {
                Console.ForegroundColor = ConsoleColor.Green;
                Console.WriteLine($"→ Il numero {estratto} è nella cartella!");
                Console.ResetColor();

                marcati[index] = true;
                numeriTrovati++;
            }

            Console.ReadKey(); // premi un tasto per continuare
        }

        // 5) TOMBOLA
        Console.ForegroundColor = ConsoleColor.Yellow;
        Console.WriteLine("\n🎉 TOMBOLA!!! 🎉");
        Console.ResetColor();
    }

    static int[] GeneraCartella(Random rnd)
    {
        int[] cartella = new int[15];
        int i = 0;

        while (i < 15)
        {
            int num = rnd.Next(1, 91);

            // Evito duplicati usando Array.IndexOf
            if (Array.IndexOf(cartella, num) == -1)
            {
                cartella[i] = num;
                i++;
            }
        }

        return cartella;
    }
}