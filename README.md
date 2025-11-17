private static BingoBall? BinarySearchBall(BingoBall[] ballArray, int value, int start, int end)
{
    while (start <= end)
    {
        int mid = (start + end) / 2;

        if (ballArray[mid].Number == value)
        {
            return ballArray[mid];
        }
        else if (ballArray[mid].Number < value)
        {
            start = mid + 1;   // cerca a destra
        }
        else
        {
            end = mid - 1;     // cerca a sinistra
        }
    }

    return null; // non trovato
}